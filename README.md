# EPA features Rancher

This tests were done with Rancher 2.6.6

To demostrate SR-IOV capapabilities and CPU-Pining/NUMA.

My setup was 
     - node-master: 1 VM with all the roles (etcd, master, worker)
     - node1 and node2: baremetal nodes with Intel Ethernet 10-Gigabit X540-AT2

if your hostnames are the same as above you could reuse all th yamls
