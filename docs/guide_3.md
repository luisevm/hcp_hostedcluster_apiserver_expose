# Expose the Hosted Cluster API Server Using a MetalLB LoadBalancer in BGP Mode

# Introduction
Exposing the API server of a Hosted Control Plane (HCP) using MetalLB in BGP mode is a robust, enterprise-grade approach. It provides highly available and load-balanced access to your hosted cluster's API server.

A key concept in HCP is that the API server of your hosted cluster runs as a pod (or three if the Hosted Cluster is configured with HA) on the management cluster. Therefore, MetalLB must be installed and configured on the management cluster, not on the hosted cluster's worker nodes.

Here is a simple guide on how to achieve this, broken down into prerequisites, FRR router setup, MetalLB configuration, hosted cluster installation, and verification steps.

MetalLB allows more advanced configurations, such as faster failure detection when a management cluster node becomes unreachable or advertising additional routes. Refer to the [OpenShift MetalLB documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/ingress_and_load_balancing/load-balancing-with-metallb#nw-metallb-l2padvertisement-cr_about-advertising-ip-address-pool) for more details.

# Prerequisites
Before deploying the hosted cluster, ensure your underlying infrastructure and management cluster are prepared:
- **Management Cluster Readiness:** You must have a management cluster with one of the following: Red Hat ACM with the Multicluster Engine operator, or a standalone OpenShift cluster with the Multicluster Engine operator.
- **MetalLB Installed:** The MetalLB Operator must be installed on the management cluster.
- **BGP-Capable Router:** A BGP-capable router configured to accept peering sessions from the management cluster nodes. In this guide, we use an FRR router installed on a RHEL host.

## Architecture

![Alt text](../images/arch_api_expose_metallb_bgp.png?raw=true "arch_api_expose_metallb_bgp")

## DNS Records and IPs

![Alt text](../images/arch_api_expose_metallb_bgp-dns.png?raw=true "arch_api_expose_metallb_bgp-dns")


# FRR Router Installation and Configuration
This section covers how to configure an external FRR router to peer with your management cluster. The ASN and neighbor IP values defined here must match the MetalLB BGP configuration applied later.

## FRR Configuration

- Install FRR

  ```sh
  sudo dnf update -y
  sudo dnf install frr -y
  ```

- Enable IP Forwarding at the OS Level

  For your Linux host to route traffic (like a real router would), you must tell the kernel to allow IP forwarding. Without this, the host will receive traffic for the MetalLB VIP but will drop it instead of routing it. Add the following line to enable IPv4 forwarding:

  Edit `/etc/sysctl.d/90-routing.conf` and add:

  ```ini
  net.ipv4.ip_forward = 1
  ```

- Reload Service

  ```sh
  sudo sysctl -p /etc/sysctl.d/90-routing.conf
  ```

- Open the Firewall for BGP

  BGP uses TCP port 179 to establish peering sessions. You need to allow this traffic so MetalLB can connect to your FRR host.
  Note: If you plan to use BFD for fast failover, you will also need to allow additional UDP ports.
  The following example uses iptables, but you can also use firewalld.

  ```sh
  sudo iptables -I INPUT -p tcp --dport 179 -j ACCEPT
  sudo iptables-save | sudo tee /etc/sysconfig/iptables
  ```

- Enable the BGP FRR Daemon

  Edit `/etc/frr/daemons` and find the `bgpd` line. Ensure it is set to `yes`:

  ```ini
  bgpd=yes
  ```

- FRR Router Configuration

  Configure FRR to establish a BGP neighbor relationship with the management cluster nodes running the MetalLB speaker pods. If the default configuration already contains a `hostname` directive, keep it. Add the following BGP configuration below it:

  Edit `/etc/frr/frr.conf`:

  ```text
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

  Start the FRR service and configure it to boot automatically if the host restarts:

  ```sh
  sudo systemctl enable --now frr
  ```

- How to Verify It's Working

  Once MetalLB is configured on your management cluster and FRR is running, check the status of the BGP sessions using the following command:

  ```sh
  sudo vtysh -c "show ip bgp summary"
  ```

  You should see your management node IPs listed, and the State/PfxRcd column should show a number (the number of routes received, which corresponds to your VIP).

> **Note:** Additional advanced configurations can be added to detect management cluster node failure and reduce the detection time by implementing BFD (Bidirectional Forwarding Detection). This isn't addressed in this repo.


# MetalLB BGP Setup

Configure MetalLB on the management cluster to allocate the VIP and advertise it via BGP.

> **Note:** The VIP (10.0.2.20) is on a different subnet than the management cluster nodes (192.168.125.x). This works because the FRR router has interfaces on both subnets and BGP advertises a /32 route for the VIP, making it reachable from the broader network.

- **IP Address Pool:** Define the pool containing your reserved VIP for the API server. The example below defines a single address, but you can configure a range of addresses.

```yaml
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

- **BGP Peer:** Establish the BGP session between your management cluster's MetalLB speaker pods and your upstream router.

```yaml
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

- **BGP Advertisement:** Tell MetalLB to advertise the API server VIP pool using the BGP protocol.

```yaml
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: hcp-api-bgp-adv
  namespace: openshift-operators
spec:
  ipAddressPools:
  - hcp-api-pool
```

With these resources in place, when the hosted cluster is created and its kube-apiserver LoadBalancer Service appears, MetalLB will assign the VIP from the pool and the BGP speakers will advertise a /32 route for that VIP to your router.


# Install the Hosted Cluster

The installation of the hosted cluster is based on a custom resource named `HostedCluster`.

> **Warning:** The configuration to expose the hosted API server must be set at installation time. It is not possible to modify the service type in Day 2.

- Go to the Red Hat ACM or OpenShift console where the Multicluster Engine operator is installed.

- Press the button to create a new cluster.

- In ACM, fill out the form to create a new cluster. Just before confirming the creation, click **Edit YAML** to switch to the YAML view and configure the service type:
  - The field that controls how the hosted API server will be exposed is under `spec.services` -> `service: APIServer`. To use a service of type LoadBalancer, apply the following configuration:

    ```yaml
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

# Verify the Connection

- **DNS:** Confirm that both API records resolve to the VIP.

  ```sh
  dig api-int.hosted.hypershift.lab +short
  dig api.hosted.hypershift.lab +short
  ```

  Both queries should return `10.0.2.20`.

- **Test connection to the hosted cluster API:** Log in using the `oc` CLI to verify end-to-end connectivity through the LoadBalancer service.

  ```sh
  oc login https://api.hosted.hypershift.lab:6443 -u kubeadmin -p <password>
  ```

  Expected output:

  ```text
  Login successful.
  You have access to 63 projects, the list has been suppressed. You can list all projects with 'oc projects'
  Using project "default".
  ```

- **Management cluster:** Verify the LoadBalancer service exposing the hosted cluster API server. The `EXTERNAL-IP` column should show the VIP from your IPAddressPool.

  The namespace follows the pattern `<clusters-namespace>-<cluster-name>`. In this example, the clusters namespace is `hosted` and the cluster name is `hosted`, resulting in `hosted-hosted`.

  ```sh
  oc -n hosted-hosted get svc kube-apiserver
  ```

  Expected output:

  ```text
  NAME             TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
  kube-apiserver   LoadBalancer   172.31.169.51   10.0.2.20     6443:32678/TCP   22m
  ```

- **FRR router:** Verify that the BGP route for the VIP has been learned. You should see a route to `10.0.2.20/32` with the management cluster nodes as next hops.

  ```sh
  ip route show 10.0.2.20
  sudo vtysh -c "show ip route"
  sudo vtysh -c "show ip bgp summary"
  sudo vtysh -c "show ip bgp"
  ```

# Troubleshooting

If something is not working as expected, check the following:

- **BGP session not establishing:** Verify that TCP port 179 is open on the FRR host and that the ASN values match between the FRR configuration and the MetalLB `BGPPeer` resource. Run `sudo vtysh -c "show ip bgp summary"` on the FRR host to check session state.
- **VIP not assigned:** Ensure the `IPAddressPool` is created in the correct namespace (`openshift-operators`) and that `autoAssign` is set to `true`. Check the MetalLB speaker pod logs on the management cluster for errors.
- **DNS not resolving:** Confirm the A records for `api.` was created and that the DNS server is reachable from the client where you are running `dig` or `oc login`.
- **Route not appearing on the FRR host:** Verify that the `BGPAdvertisement` resource references the correct `ipAddressPools` name and that the BGP sessions are in `Established` state.

# Summary

This guide walked through the end-to-end process of exposing a hosted cluster's API server using a MetalLB LoadBalancer service with BGP:

1. An FRR router was configured to peer with the management cluster nodes.
2. MetalLB was set up on the management cluster with an IP address pool, BGP peer, and advertisement.
3. The hosted cluster was created with the `LoadBalancer` service publishing strategy.
4. DNS records were configured to point to the VIP, and connectivity was verified.

For next steps, consider configuring ingress for the hosted cluster's application routes or adding BFD for faster failover detection.
