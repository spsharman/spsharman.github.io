# Configuring route leaking between OSPF and BGP

These are the basic configuration steps that I put together to peer a csr1kv to ACI such that the csr1kv provides routing between "shared-services" tenant running OSPF and a tenant running BGP.

<div class="row" style="display: table;margin: 0 auto">
    <img src="./images/1.png" width="800" >
</div>

The steps below focus only on the csr1kv configuration.

## Part 1 - Basic configuration

I added a dedicated management VRF on the csr1kv just so that I didn't cut my legs off when playing with different configuration options.

```console title="Management VRF"
vrf definition management
 
 address-family ipv4
 exit-address-family

interface GigabitEthernet1
 vrf forwarding management
 ip address 10.237.100.127 255.255.255.0
 negotiation auto
 no mop enabled
 no mop sysid

ip route vrf management 0.0.0.0 0.0.0.0 10.237.100.1
```

I then configured OSPF on the csr1kv to peer with my upstream routers

In this case the upstream routers are my ACI Border Leafs and I'm peering OSPF with a L3out SVI in my shared-services tenant (shared-services:vrf-01).

```console title="OSPF Configuration"
interface GigabitEthernet2
 ip address 10.237.99.52 255.255.255.248
 negotiation auto
 no mop enabled
 no mop sysid

router ospf 1
 network 10.237.99.48 0.0.0.7 area 0.0.0.1
```

At this point my OSPF neighbours we up and working and I was receiving routes.

```console title="OSPF up and running"
csr1kv-02#show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
101.2.1.1         1   FULL/BDR        00:00:34    10.237.99.49    GigabitEthernet2
102.2.1.1         1   FULL/DR         00:00:33    10.237.99.50    GigabitEthernet2

csr1kv-02#sh ip route

Routing Table: 

Gateway of last resort is 10.237.99.50 to network 0.0.0.0

O*E2    0.0.0.0/0   [110/1] via 10.237.99.50, 21:44:43, GigabitEthernet2
                    [110/1] via 10.237.99.49, 21:44:43, GigabitEthernet2
        10.0.0.0/8 is variably subnetted, 78 subnets, 9 masks
O E2    10.0.1.0/24 [110/20] via 10.237.99.50, 21:44:43, GigabitEthernet2
                    [110/20] via 10.237.99.49, 21:44:43, GigabitEthernet2
O E2    10.0.2.0/24 [110/20] via 10.237.99.50, 21:44:43, GigabitEthernet2
                    [110/20] via 10.237.99.49, 21:44:43, GigabitEthernet2
O E2    10.0.3.0/24 [110/20] via 10.237.99.50, 21:44:43, GigabitEthernet2
                    [110/20] via 10.237.99.49, 21:44:43, GigabitEthernet2
O E2    10.0.4.0/24 [110/20] via 10.237.99.50, 21:44:43, GigabitEthernet2

<output truncated>
```

The next thing to configure was BGP peering from the csr1kv to the L3out SVI in my demo-05 tenant (demo-05:vrf-01) and redistribute the routes from OSPF.

```console title="BGP Configuration"
interface GigabitEthernet5
 ip address 10.237.99.60 255.255.255.248
 negotiation auto
 no mop enabled
 no mop sysid

router bgp 65051
 bgp log-neighbor-changes
 neighbor 10.237.99.57 remote-as 65151
 neighbor 10.237.99.58 remote-as 65151
 
 address-family ipv4
  network 0.0.0.0
  network 10.237.99.56 mask 255.255.255.248
  redistribute ospf 1 match external 2
  neighbor 10.237.99.57 activate
  neighbor 10.237.99.58 activate
 exit-address-family
```

In this example I used the ```network 0.0.0.0``` statement which will look in the routing table for a default route and then advertise in BGP. 

Note: If a default route has been learned as an external route from OSPF it will **not** be redistributed into BGP using ```redistribute ospf 1 match external 2```.

Another way to advertise a default route would be to use ```default-information originate``` with ```neighbor x.x.x.x default-originate``` this would always create a default route and advertise it via BGP to the specified neighbours.

```console
router bgp 65051
 bgp log-neighbor-changes
 neighbor 10.237.99.57 remote-as 65151
 neighbor 10.237.99.58 remote-as 65151

 address-family ipv4
  network 10.237.99.56 mask 255.255.255.248
  neighbor 10.237.99.57 activate
  neighbor 10.237.99.57 default-originate
  neighbor 10.237.99.58 activate
  neighbor 10.237.99.58 default-originate
  default-information originate
 exit-address-family
```

## Need to add redistribution from BGP back to OSPF

At this point my BGP neighbours we up and working and I was receiving routes.

```console title="BGP up and running"
csr1kv-02#show ip ospf neighbor
```

## Part 2 - Creating a Tenant specific VRF on the upstream router

Once everything was working in the global routing table I created a dedicated Tenant VRF on the csr1kv so that routes from different ACI tenants would remain seperated.

The first step was to define a new default VRF and a new Tenant specific VRF:

```console title="VRF definitions"
vrf definition default
 rd 65000:100
 route-target export 65000:100
 route-target import 65000:100
 route-target import 65005:5

 address-family ipv4
 exit-address-family

vrf definition tn-demo-05
 rd 65005:5
 route-target export 65005:5
 route-target import 65005:5
 route-target import 65000:100

 address-family ipv4
 exit-address-family
```

Then I configured the interfaces:

```console title="Interface configuration"
interface GigabitEthernet2
 vrf forwarding default
 ip address 10.237.99.52 255.255.255.248
 negotiation auto
 no mop enabled
 no mop sysid

interface GigabitEthernet5
 vrf forwarding tn-demo-05
 ip address 10.237.99.60 255.255.255.248
 negotiation auto
 no mop enabled
 no mop sysid
```

The final step was to configure the routing processes:

```console title="Configure the routing processes"
router ospf 1 vrf default
 capability vrf-lite
 network 10.237.99.48 0.0.0.7 area 0.0.0.1

router bgp 65051
 bgp log-neighbor-changes

 address-family ipv4 vrf default
  redistribute connected
  redistribute ospf 1 match internal external 2
 exit-address-family

 address-family ipv4 vrf tn-demo-05
  network 0.0.0.0
  network 10.237.99.56 mask 255.255.255.248
  neighbor 10.237.99.57 remote-as 65151
  neighbor 10.237.99.57 activate
  neighbor 10.237.99.58 remote-as 65151
  neighbor 10.237.99.58 activate
```
