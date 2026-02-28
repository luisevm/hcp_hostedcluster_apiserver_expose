# Publish Hosted Cluster API with LoadBalancer Service

# Introduction
Exposing the API server of a Hosted Control Plane (HCP) using MetalLB in BGP mode is a robust, enterprise-grade approach. It provides highly available, and round robin access to your backplane control plane components.

An important concept of HCP is understand that the API server of your hosted cluster runs as a pod (ot three if Hosted Cluster is configured with HA) on the Management cluster. Therefore, MetalLB must be installed and configured on the management cluster, not on the hosted cluster's worker nodes.

Here is a simple guide on how to achieve this, broken down into prerequisites, installation parameters, Day 2 configuration, and other architectural considerations.

MetalLB allows more advanced configurations such as to dectect faster when a backplane node is unreachable or configure additional routes to be advertized. Refer to the OpenShift MetalLB documentation for more details, [here](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/ingress_and_load_balancing/load-balancing-with-metallb#nw-metallb-l2padvertisement-cr_about-advertising-ip-address-pool)

# Prerequisites to Cope with the Requirements
Before deploying the hosted cluster, ensure your underlying infrastructure and management cluster are prepared:
- Management Cluster Readiness: You must have a management cluster, with Red Hat ACM with Multicluster engine operator installed or OpenShift cluster with Multicluster engine operator installed.
- MetalLB Installed: The MetalLB Operator must be installed on the management cluster.
- Router BGP-capable configured to accept peering from the management cluster nodes. In this repo I will use FRR router and install it in RHEL.

## Architecture

![Alt text](../images/arch_api_expose_metallb_bgp.png?raw=true "arch_api_expose_metallb_bgp")

## DNS records and IPs

![Alt text](../images/arch_api_expose_metallb_bgp-dns.png?raw=true "arch_api_expose_metallb_bgp-dns")


# FRR Router Installation and Configuration
Here is how you configure an external FRR router to peer with your management cluster, and how this impacts your MetalLB configuration.

## FRR Configuration

- Install FRR

  ```sh
  sudo dnf update -y
  sudo dnf install frr -y
  ```

- Enable IP Forwarding at the OS Level
For your Fedora VM to route traffic (like a real router would), you must tell the Linux kernel to allow IP forwarding. Without this, your VM will receive traffic for the MetalLB VIP but will drop it instead of routing it.
Add the following line to enable IPv4 forwarding:

  ```sh
  sudo vi /etc/sysctl.d/90-routing.conf
  net.ipv4.ip_forward = 1
  ```

- Reload Service

  ```sh
  sudo sysctl -p /etc/sysctl.d/90-routing.conf
  ```

- Open the Firewall for BGP
BGP uses TCP port 179 to establish peering sessions. You need to allow this traffic so MetalLB can connect to your FRR VM.
Note: If you plan to use BFD for fast failover as mentioned previously, you will also need to allow additional UDP ports.
Configuring iptables, but can use firewalld.

  ```sh
  sudo iptables -I INPUT -p tcp --dport 179 -j ACCEPT
  sudo iptables-save | sudo tee /etc/sysconfig/iptables
  ```

- Enable the BGP FRR Daemon

  ```sh
  sudo vi /etc/frr/daemons
  Find the lines bgpd and ensure they are set to yes:
  
  bgpd=yes
  ```

- FRR Router Configuration
To make FRR talk to MetalLB, you need to configure FRR to establish a BGP neighbor relationship with the management cluster nodes running the MetalLB speaker pods. If the configuration contains a hostname, keep it, and add the following in the next line:

```sh
sudo vi /etc/frr/frr.conf

frr defaults traditional
log syslog informational

router bgp 64512
  bgp router-id 192.168.125.1
  no bgp ebgp-requires-policy

  ! Define your MetalLB nodes
  neighbor 192.168.125.20 remote-as 64513
  neighbor 192.168.125.21 remote-as 64513
  neighbor 192.168.125.22 remote-as 64513

  address-family ipv4 unicast
    neighbor 192.168.125.20 activate
    neighbor 192.168.125.21 activate
    neighbor 192.168.125.22 activate
  exit-address-family
```

- Start and Enable the FRR Service

- Finally, start the FRR service and configure it to boot automatically if the hosting OS restarts.
sudo systemctl enable --now frr

- How to Verify It's Working:
Once MetalLB is configured on your management cluster and FRR is running, check the status of the BGP sessions using the following command:

  ```sh
  sudo vtysh -c "show ip bgp summary"
  ```
  You should see your management node IPs listed, and the State/PfxRcd column should show a number (the number of routes received, which corresponds to your VIP).

Additional advanced configurations can be added to detect backplane failure, to reduce the failure detection time by implementing BFD (Bidirectional Forwarding Detection). This isnt addressed in this repo.


# MetalLB BGP Setup

Configure MetalLB on the management cluster to allocate the VIP and advertise it via BGP.

- IP Address Pool Define the pool containing your reserved VIP for the API server.
Here I defined only one address but one can configure a range of adresses.

```sh
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: hcp-api-pool
  namespace: openshift-operators
spec:
  addresses:
  - 10.0.2.20-10.0.2.20
  autoAssign: true
```

- BGP Peer Establish the BGP session between your management cluster's MetalLB speaker pods and your upstream router.

```sh
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: upstream-router-peer
  namespace: openshift-operators
spec:
  peerAddress: 192.168.125.1
  peerASN: 64512
  myASN: 64513
```

- BGP Advertisement Tell MetalLB to advertise the API server VIP pool using the BGP protocol.

```sh
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: hcp-api-bgp-adv
  namespace: openshift-operators
spec:
  ipAddressPools:
  - hcp-api-pool
```

Once these are applied, MetalLB will assign the VIP from the pool to the kube-apiserver LoadBalancer service, and the BGP speakers will advertise a /32 route for that VIP to your router.


# Install the Hosted Cluster
- The installation of the hosted cluster is based on a custom resource named HostedCluster and the configuration to expose the hosted apiserver must be done in installation time, and its not possible to modify the service type in Day2.

- Go the RH ACM or Openshift instance where Multicluster operator is installed.

- And press the button to create a new cluster

- In ACM configure the formulary to create a new cluster. Just before acknoledging to create a new cluster, press to view the YAML and configure the service type:
  - The part that controls how the hosted apiserver will be exposed is under spec.services -> "service: APIServer". And to use a service type LoadBalancer the following must be the configuration.

    ```sh
    apiVersion: hypershift.openshift.io/v1beta1
    kind: HostedCluster
    ...
    spec:
    ...
      services:
      - service: APIServer
        servicePublishingStrategy:
          type: LoadBalancer
    ```

# Verify the connection

- DNS

```sh
dig api-int.hosted.hypershift.lab +short
dig api.hosted.hypershift.lab +short
10.0.2.20
10.0.2.20
```

- Test connection to the hosted cluster api

```sh
oc login https://api.hosted.hypershift.lab:6443 -u kubeadmin -p im2xc-kA38e-FpSdU-hf398

---output---
Login successful.
You have access to 63 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
[lab-user@bastion ~]$ 
```

- Management cluster - service exposing hosted cluster apiserver

  ```sh
  oc -n hosted-hosted get svc kube-apiserver

  NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
  kube-apiserver                       LoadBalancer   172.31.169.51    10.0.2.20     6443:32678/TCP      22m
  ```

- On FRR router

```sh
ip route show 10.0.2.20
sudo vtysh -c "show ip route"
sudo vtysh -c "show ip bgp summary"
sudo vtysh -c "show ip bgp"
```


