# Container Networking Talk Notes

* I work for Oracle, and Oracle has a managed Kubernetes service, and some time
  back I was given the task of looking into updating the networking layer in
  this service from using Flannel (an overlay network) to a solution which
  utilises the native networking features of the Oracle cloud (secondary VNICs +
  IPs). Don't worry if you dont know what Flannel is, or know what an overlay
  network is, as that is the point of this talk!  However, once I started digging
  in, I quickly found that I didn't understand how Flannel worked, and it seemed
  a little wrong to replace one thing with another solution, if you dont
  understand how the original worked. So, I started digging deeper, and then soon
  realised that I don't understand networking in general!  Long story short, big
  rabbit hole, learnt some stuff, and most importantly, found that I really
  enjoyed this area, so I thought I would write a talk and come and spread the
  networking love!
  
* So, I'm Kris, and in the next 30 minutes or so, I'm going to attempt to explain
  how a container on one computer on the internet, can connect to a container on 
  another computer, somewhere else on the internet.

## Slide: The aim

* Aim to use the Kubernetes model of networking.
    
1. Each container (pod) has its own unique IP. 
2. No NAT'ing going on.
3. Host can talk to containers, and vice versa.

## Slide: The plan

* Going to work our way toward the general case in 4 steps.

* For each, we will explain the model via a diagram. Show some code, run the code, 
  then test what we have created. 

* Each step we be created using vagrant based VMs.

* Summarise the 4 steps.

## Slide: Single network namespace diagram

* Describe the outer box (the node). Could be a physical machine, or a VM as in this case.

* Describe containers vs namespaces: Containers use a bunch of different Linux mechanisms 
  to isolate the processes running inside, both in terms of system calls, available resources, 
  what it can see, i.e. filesystems, other processes, etc. However, from a network connectivity 
  point of view, the only mechanism that matters here is the network namespace, so from now on, 
  whenever I say container, what I really mean is network namespace.

* What is a network namespace: It's another instance of the kernels network stack containing: 
    1. It's own interfaces.
    2. It's own routing + route tables.
    3. It's own IPtables rules.
        
* When created, it is empty, i.e. no interfaces, routing or IP tables rules.

* Describe VETH pair: Ethernet cable with NIC on each end.

* Describe the relevant routing from/to the network namespace:
    1. Directly connected route from the host to the network namespace.
    2. Default route out of the network namespace.

* Note The 'aha' moment, when I worked out the possible types of routing rules. 
  For me, understanding these was for me the key to understanding networking in general.

## Code: Single network namespace setup.sh

* Explain that we are running in a single node vagrant setup.
* Open the the *setup.sh*.
* Talk about the *ip* tool.
* Give a quick run through of each line.

## Demo: Single network namespace

```
./setup.sh
ip netns
sudo ip netns exec con ip a
ping 176.16.0.1
```

* What is actually responding to the pings in these cases, as there is no process running 
  inside the namespace who can respond in this case? It is the kernel network stack 
  inside of the network namespace that is responding to these IMCP echo requests, with 
  an ICMP echo request packet.

* For a more realistic example, We would run one (or more) real process in the network namespace. 
  However, for the purposes of testing connectivity, pinging is enough.

* Note: you can run multiple processes inside a network namespace, which is what happens 
  inside Kubernetes pods.

## Slide: Diagram of multiple network namespaces on the same node

* Describe the Linux bridge: It is a virtual ethernet switch (i.e. a single L2 broadcaset domain), 
  implemented in the kernel.
* The bridge has its own subnet.
* The bridge also has its own IP. This allows access from the outside.
* Describe the route for the subnet.
* Note: This corresponds to the default *docker0* bridge.

## Code: Multiple network namespace setup.sh

* Explain that we now using a (new but similar) single node vagrant environment.
* Open the *setup.sh*.
* Describe the parts common to the previous step.
* Describe the bridge creation lines.

## Demo: Multiple network namespace

```
./setup.sh
ip a
sudo ip netns exec con1 ping 172.16.0.3
sudo ip netns exec con1 ping 10.0.0.10
```

* Highlight the TTL. Should be the default value, thus no routing is going on here!
* Describe what the TTL is, and what happens when the TTL reaches zero.

## Slide: Diagram of multiple network namespaces on different nodes but same L2 network

* First do a quick recap of what we mean when we say 'L2' network (switches only, single broadcast domain), 
  and contrast this with an 'L3' network (switches + routers, multiple broadcast domains).
* 2 nodes on the same subnet, each setup the same as 2 but with containing different network namespace subnets.
* Talk about the routing within the node. 
* Talk about the (next hop) routing between nodes (only works if the nodes are on the same L2 network). 
* Note that this is how the *host-gw* flannel backend works, and also single L2 *Calico*.

## Code: Multi node setup.sh

* Explain that we are now using a 2 node vagrant setup.
* Open the *setup.sh*.
* Describe the parts common to the previous step.
* Describe the setup of the extra routes.
* Explain the IP forwarding: Turns your Linux box into a router, which is 
  required in this case as the node has to forward the packets for any network 
  namespaces that live on that node.

## Demo: Multi node

On each node, run:

```
./setup.sh
```

And again on each node, run:

```
ip r
```

From 10.0.0.10:

```
sudo ip netns exec con1 ping 172.16.1.2
sudo ip netns exec con1 ping 10.0.0.20
```

* When we ping from a network namespaces to another network namespace across nodes, 
  highlight the TTL. Explain the reported value.

* When we ping a network namespace on the other node from the node, 
  highlight the TTL. Explain the reported value.

## Slide: Diagram of multiple network namespaces on different nodes on different L2 networks (the overlay network)

* Now can't use static routes, as nodes could be on different subnets. Options:
    1. Update routes on all routers in between (which can he done if you have control over the routers).
    2. If running on cloud, then they might provide an option to add routes (node-\>pod-subnet mappings) into your virtual network. For example, AWS (and Oracle cloud) both allow this.
    3. Another way us to use overlay network, which is what we will describe here.
* Introduce *tun* devices. A network interface backed by a user-space process.
* A *tun* device accepts/outputs raw IP packets.  
* How would we use it in this case.
* Now no need for the static routes.

## Slide: Diagram of the route of a packet through the tun devicies

* Talk about the routing for the overlay.
* This corresponds to the UDP backend for flannel (only recommended for debugging).
* For production, the *VXLAN* backend is recommended.
* But isnt UDP unreliable? Yes, but we are sending a reliable stream over an unreliable medium,
  which isnt really all that different from sending it over the wire/ethernet, which is also unreliable..

## Code: Overlay network setup.sh

* Explain that we are now using a (new but similar) 2 node vagrant setup.
* Talk through the *setup.sh*. 
* Describe (briefly) the parts common to the previous step.
* We still need IP forwarding enabled here. This allows the node to act as a router, i.e.
  to accept and forward packets received, but not destined for, the IP of the node.
* Now no extra routes, but contains the *socat* implementation of the overlay.
* Describe *socat* in general. It creates 2 bidirectional bytestreams, and transfers data between them.
* Describe how *socat* is being used here. 
* Note the MTU settings, what is going on here? We reduce the MTU of the *tun0* 
  device as this allows for the 8 bytes UDP header that will be added, thus ensuring that 
  fragmentation does not occur.
* Reverse packet filtering: What is this: Discards incoming packets from interfaces where they shouldn't be.
* It's purpose: A security feature to stop IP spoofed packets from being propagated.
* Why do we need the reverse packet filtering in this case? Consider the case where we send
  a packet from a node to a container on the other node. The outward packet will go over the 
  tunnel. However, the response will not (as it is destined for the node), thus the response
  will emerge on a different interface to which the request packet went. Therefore, the kernel 
  consider this suspicious, unless we tell it that all is ok.

## Demo: Overlay network

On each node, run:

```
./setup.sh
```

From 10.0.0.10:

```
sudo ip netns exec con1 ping 172.16.1.2
sudo ip netns exec con1 ping 10.0.0.20
```

* When we ping from a network namespace to a network namespace across nodes, 
  highlight the TTL. Explain the reported value (should have decreased by 2).

* When we ping from a node to a remote network namespace, 
  highlight the TTL. Explain the reported value (should have decreased by 1).

To see the encapsulation process more clearly:

On node 10.0.0.10:

```
./send.sh
```

Meanwhile, on node 10.0.0.20:

```
./capture.sh enp0s8
./capture.sh tun0
./capture.sh br0
```

## Slide: Recap

* Describe the stages we have covered.
* Describe the key takeaways (concepts and tools).

## Slide: Putting it all together

* So how does this work in the real world?

* Can characterise existing Kubernetes networking solutions in terms of 
  2 properties. 1. How they connect, and 2. Where they store their pod-subnet
  to node mappings.

* Popular network solutions:
    * 1. *Flannel* 
        * Multiple backends:
            * *host-gw*: step 3
            * *udp*: step 4
            * *VXLAN*: step 4, but more efficient. 
            * *awsvpc*: Sets routes in AWS.
        * Uses *etcd* to store the node->pod-subnet mapping.
    * 2. *Calico*
        * No overlay for intra L2. Uses next-hop routing (step 3).
        * For inter L2 node comminucation, uses IPIP overlay.
        * Node->pod-subnet mappings distributed to nodes using BGP.
    * 3. *Weave*
        * Similar to *Flannel*, uses *VXLAN* overlay for connectivity.
        * No need for *etcd*. Node->pod-subnet mapping distrubuted to each node via gossiping. 

## Slide: Github

All this is available on GitHub *kristenjacobs/container-networking*

## Slide: Questions?
