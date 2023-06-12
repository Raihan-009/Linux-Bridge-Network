# Linux-Bridge-Network



## Step 1: Create a Linux bridge:

```bash
sudo ip link add dev v-net type bridge
```
Lets run `sudo ip links show` to check and he ***expected output*** might look like

```shell
12: v-net: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether d2:22:49:28:a4:c8 brd ff:ff:ff:ff:ff:ff
```
But here `v-net` is now in down state. So lets turn into **up**.

```shell
sudo ip link set dev v-net up
```
and now ***expected output*** might look like
```bash
12: v-net: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default qlen 1000
    link/ether d2:22:49:28:a4:c8 brd ff:ff:ff:ff:ff:ff
```
Here, the "UP" flag indicates that the interface is enabled and operational, while the "DOWN" state indicates that the interface is currently inactive or not functioning as there is no any physical connectivity of the interface right now.

## Step 2: Assign an IP address to the bridge interface 'v-net':

```shell
sudo ip address add 10.0.0.1/24 dev v-net
```
Run `sudo ip addr show dev v-net`

```bash
12: v-net: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether d2:22:49:28:a4:c8 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.1/24 scope global v-net
       valid_lft forever preferred_lft forever
```

## Step 3: Create three network namespaces:

```shell
sudo ip netns add blue-ns
sudo ip netns add gray-ns
sudo ip netns add lime-ns
```
Run `sudo ip netns list` to check the list of namespaces.

## Step 4: Create virtual Ethernet pairs:
```shell
sudo ip link add veth-blue-ns type veth peer name veth-blue-br
sudo ip link add veth-gray-ns type veth peer name veth-gray-br
sudo ip link add veth-lime-ns type veth peer name veth-lime-br
```
Each cable now has two ends. Before connecting them to the appropriate bridge and namespaces, run `sudo ip link show` command to verify if the cables have been successfully created.
***Expected Output***
```bash
13: veth-blue-br@veth-blue-ns: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether fa:b8:a7:3c:41:0a brd ff:ff:ff:ff:ff:ff
14: veth-blue-ns@veth-blue-br: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether ea:ff:ec:28:f3:aa brd ff:ff:ff:ff:ff:ff
15: veth-gray-br@veth-gray-ns: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether be:2a:cb:68:4d:18 brd ff:ff:ff:ff:ff:ff
16: veth-gray-ns@veth-gray-br: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 1a:08:51:d7:ab:95 brd ff:ff:ff:ff:ff:ff
17: veth-lime-br@veth-lime-ns: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 02:9a:fd:3e:85:8a brd ff:ff:ff:ff:ff:ff
18: veth-lime-ns@veth-lime-br: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether fe:c0:37:07:25:e4 brd ff:ff:ff:ff:ff:ff
```

## Step 5: Move each end of veth cable to a different namespace:
```shell
sudo ip link set dev veth-blue-ns netns blue-ns
sudo ip link set dev veth-gray-ns netns gray-ns
sudo ip link set dev veth-lime-ns netns lime-ns
```

Run `sudo ip netns exec <namespace-name> ip link show` to verify

## Step 6: Add the other end of the virtual interfaces to the bridge:
```shell
sudo ip link set dev veth-blue-br master v-net
sudo ip link set dev veth-gray-br master v-net
sudo ip link set dev veth-lime-br master v-net
```
Again run `sudo ip link show` command to verify.

## Step 7: Set the bridge interfaces up:
```shell
sudo ip link set dev veth-blue-br up
sudo ip link set dev veth-gray-br up
sudo ip link set dev veth-lime-br up
```
Run `sudo ip link show` and ***expected output*** might look like
```bash
13: veth-blue-br@if14: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue master v-net state LOWERLAYERDOWN mode DEFAULT group default qlen 1000
    link/ether fa:b8:a7:3c:41:0a brd ff:ff:ff:ff:ff:ff link-netns blue-ns
15: veth-gray-br@if16: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue master v-net state LOWERLAYERDOWN mode DEFAULT group default qlen 1000
    link/ether be:2a:cb:68:4d:18 brd ff:ff:ff:ff:ff:ff link-netns gray-ns
17: veth-lime-br@if18: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue master v-net state LOWERLAYERDOWN mode DEFAULT group default qlen 1000
    link/ether 02:9a:fd:3e:85:8a brd ff:ff:ff:ff:ff:ff link-netns lime-ns
```
The "LOWERLAYERDOWN" state typically applies to virtual interfaces that are dependent on another interface or component. If one end is not currently in the 'up' state, it suggests that there may be an issue with the namespaces at that end (in our case).

## Step 8: Set the namespace interfaces up:
```bash
sudo ip netns exec blue-ns ip link set dev veth-blue-ns up
sudo ip netns exec gray-ns ip link set dev veth-gray-ns up
sudo ip netns exec lime-ns ip link set dev veth-lime-ns up
```
Now, run `sudo ip link show` command again and it should show all interfaces are currently in UP state.

## Step 9: Assign IP addresses to the virtual interfaces within each namespace and set the default routes:
- In the blue-ns namespace:
  ```bash
  sudo ip netns exec blue-ns ip address add 10.0.0.11/24 dev veth-blue-ns
  sudo ip netns exec blue-ns ip route add default via 10.0.0.1
  ```
- In the gray-ns namespace:
  ```bash
  sudo ip netns exec gray-ns ip address add 10.0.0.21/24 dev veth-gray-ns
  sudo ip netns exec gray-ns ip route add default via 10.0.0.1
  ```
- In the lime-ns namespace:
  ```bash
  sudo ip netns exec lime-ns ip address add 10.0.0.31/24 dev veth-lime-ns
  sudo ip netns exec lime-ns ip route add default via 10.0.0.1
  ```