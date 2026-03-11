# kube-ovn-experiments

For `02-underlay`, to ping from lab bare-metal nodes, need to add.
```bash
ip route add 129.97.168.0/24 via 10.10.10.1 dev net1
```


# Kube-OVN Custom VPC with NAT Gateway and Floating IP

This example demonstrates how to expose a Kubernetes pod through a
**Kube-OVN custom VPC** using:

-   a **secondary Multus network**
-   a **VPC NAT gateway**
-   a **floating IP (FIP)**

The result is that external hosts can reach a pod using a **stable
external IP**, while the pod still retains normal Kubernetes networking.

------------------------------------------------------------------------

# Architecture Overview

The deployment creates a **tenant VPC (`red-vpc`)** with two subnets:

  Subnet            Purpose                    Example
  ----------------- -------------------------- ---------
  `172.31.0.0/24`   Primary pod network        `eth0`
  `172.30.0.0/24`   Secondary attach network   `net1`

The attach network connects pods to a **NAT gateway**, which exposes
them to an external subnet.

                           External Network
                           10.10.10.0/24
                                 │
                                 │
                           Floating IP
                            10.10.10.82
                                 │
                                 ▼
                       ┌───────────────────┐
                       │   NAT Gateway     │
                       │                   │
                       │  WAN: 10.10.10.x  │
                       │  LAN: 172.30.0.254│
                       └─────────┬─────────┘
                                 │
                                 │
                        attachnet-red subnet
                           172.30.0.0/24
                                 │
                                 │
                            net1 │
                           ┌─────▼─────┐
                           │  red-pod  │
                           │           │
                           │ net1:     │
                           │ 172.30.0.2│
                           │           │
                           │ eth0:     │
                           │ 172.31.0.x│
                           └─────┬─────┘
                                 │
                         Kubernetes overlay
                            172.31.0.0/24

------------------------------------------------------------------------

# Pod Network Interfaces

Each pod has **two interfaces**:

  -----------------------------------------------------------------------
  Interface               Network                 Purpose
  ----------------------- ----------------------- -----------------------
  `eth0`                  Kube-OVN primary        Kubernetes
                          network                 control-plane
                                                  connectivity

  `net1`                  attach network          data-plane traffic via
                                                  NAT gateway
  -----------------------------------------------------------------------

Example inside the pod:

``` bash
ip addr
```

    eth0   172.31.0.2/24
    net1   172.30.0.2/24

------------------------------------------------------------------------

# NAT Gateway

The `VpcNatGateway` acts as a router between the tenant VPC and the
external network.

  Interface   Network
  ----------- ----------------
  LAN         `172.30.0.254`
  WAN         `10.10.10.x`

Responsibilities:

-   performs **SNAT** for outbound traffic
-   performs **DNAT** for floating IP rules
-   routes traffic between the VPC and external subnet

------------------------------------------------------------------------

# Floating IP Mapping

A floating IP maps an external address to an internal pod address.

    10.10.10.82  <->  172.30.0.2

Traffic flow:

    external host
    129.97.168.100
            │
            ▼
    10.10.10.82 (floating IP)
            │
            ▼
    NAT gateway
            │
    DNAT
            ▼
    172.30.0.2
            │
            ▼
    red-pod

------------------------------------------------------------------------

# Pod Routing Configuration

The pod must send external traffic through its **secondary interface
(`net1`)** rather than the Kubernetes interface (`eth0`).

Instead of replacing the default route, we use **policy routing**.

Init container configuration:

``` bash
ip rule add from 172.30.0.2 table 100
ip route add default via 172.30.0.254 dev net1 table 100
```

Result:

  Traffic Source              Route
  --------------------------- ----------------------
  Kubernetes control-plane    `eth0`
  Traffic from `172.30.0.2`   `net1 → NAT gateway`

Routing table example:

``` bash
ip rule
```

    0: from all lookup local
    32765: from 172.30.0.2 lookup 100
    32766: from all lookup main

Routing table 100:

``` bash
ip route show table 100
```

    default via 172.30.0.254 dev net1

------------------------------------------------------------------------

# Packet Flow

## Incoming Traffic

    client
    129.97.168.100
          │
          ▼
    10.10.10.82 (Floating IP)
          │
          ▼
    NAT Gateway
          │
    DNAT
          ▼
    172.30.0.2
          │
          ▼
    red-pod net1

------------------------------------------------------------------------

## Return Traffic

    red-pod
    source = 172.30.0.2
          │
          ▼
    policy routing table 100
          │
          ▼
    172.30.0.254 (NAT gateway)
          │
    SNAT
          ▼
    10.10.10.82
          │
          ▼
    external client

------------------------------------------------------------------------

# Key Design Principles

### 1. Primary interface remains intact

`eth0` remains connected to Kubernetes networking so that:

-   DNS works
-   services work
-   control plane connectivity is preserved

------------------------------------------------------------------------

### 2. Secondary interface handles data-plane traffic

The `net1` interface allows:

-   custom VPC routing
-   NAT gateway connectivity
-   external exposure

------------------------------------------------------------------------

### 3. Policy routing prevents routing conflicts

Instead of changing the default route, traffic is selected based on
**source IP**.

This allows both networks to coexist safely.

------------------------------------------------------------------------

# Useful Debug Commands

Check pod networking:

``` bash
kubectl exec -n red red-pod -- ip addr
kubectl exec -n red red-pod -- ip rule
kubectl exec -n red red-pod -- ip route show table 100
```

Check NAT gateway:

``` bash
kubectl get pods -n kube-system | grep nat
kubectl exec -n kube-system vpc-nat-gw-red-gw-0 -- iptables -t nat -L
```

Check floating IP:

``` bash
kubectl get iptables-eip
kubectl get iptables-fip
```