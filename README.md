# kube-ovn-experiments

For `02-underlay`, to ping from lab bare-metal nodes, need to add.
```bash
ip route add 129.97.168.0/24 via 10.10.10.1 dev net1
```
