//membuat vpc network

- gcloud compute networks create vpc-demo --subnet-mode custom

//create subnet vpc-network us-central

- gcloud compute networks subnets create vpc-demo-subnet1 \
  --network vpc-demo --range 10.1.1.0/24 --region us-central1

//create subnet vpc-network us-east

- gcloud compute networks subnets create vpc-demo-subnet2 \
  --network vpc-demo --range 10.2.1.0/24 --region us-east1

//Create a firewall rule to allow all custom traffic

- gcloud compute firewall-rules create vpc-demo-allow-custom \
  --network vpc-demo \
  --allow tcp:0-65535,udp:0-65535,icmp \
  --source-ranges 10.0.0.0/8

//Create a firewall rule to allow SSH, ICMP traffic from anywhere

- gcloud compute firewall-rules create vpc-demo-allow-ssh-icmp \
   --network vpc-demo \
   --allow tcp:22,icmp

//Create a VM instance vpc-demo-instance1 in zone us-central1-b

- gcloud compute instances create vpc-demo-instance1 --zone us-central1-b --subnet vpc-demo-subnet1

//Create a VM instance vpc-demo-instance2 in zone us-east1-b

- gcloud compute instances create vpc-demo-instance2 --zone us-east1-b --subnet vpc-demo-subnet2

//create a VPC network called on-prem (premises)

- gcloud compute networks create on-prem --subnet-mode custom

//Create a subnet called on-prem-subnet1

- gcloud compute networks subnets create on-prem-subnet1 \
  --network on-prem --range 192.168.1.0/24 --region us-central1

//Create a firewall rule to allow all custom traffic within the network:

- gcloud compute firewall-rules create on-prem-allow-custom \
  --network on-prem \
  --allow tcp:0-65535,udp:0-65535,icmp \
  --source-ranges 192.168.0.0/16

//Create a firewall rule to allow SSH, RDP, HTTP, and ICMP traffic to the instances:

- gcloud compute firewall-rules create on-prem-allow-ssh-icmp \
   --network on-prem \
   --allow tcp:22,icmp

Create an instance called on-prem-instance1 in the region us-central1:

- gcloud compute instances create on-prem-instance1 --zone us-central1-a --subnet on-prem-subnet1

//create an HA VPN in the vpc-demo network:

- gcloud compute vpn-gateways create vpc-demo-vpn-gw1 --network vpc-demo --region us-central1

//Create an HA VPN in the on-prem network

- gcloud compute vpn-gateways create on-prem-vpn-gw1 --network on-prem --region us-central1

//Create an HA VPN in the on-prem network

- gcloud compute vpn-gateways create on-prem-vpn-gw1 --network on-prem --region us-central1

//View details of the vpc-demo-vpn-gw1 gateway to verify its settings:

- gcloud compute vpn-gateways describe vpc-demo-vpn-gw1 --region us-central1

//View details of the on-prem-vpn-gw1 vpn-gateway to verify its settings:

- gcloud compute vpn-gateways describe on-prem-vpn-gw1 --region us-central1

-- Create cloud routers --

//Create a cloud router in the vpc-demo network:

- gcloud compute routers create vpc-demo-router1 \
   --region us-central1 \
   --network vpc-demo \
   --asn 65001

//Create a cloud router in the on-prem network:

- gcloud compute routers create on-prem-router1 \
   --region us-central1 \
   --network on-prem \
   --asn 65002

--Create two VPN tunnels--

//Create the first VPN tunnel in the vpc-demo network:

- gcloud compute vpn-tunnels create vpc-demo-tunnel0 \
   --peer-gcp-gateway on-prem-vpn-gw1 \
   --region us-central1 \
   --ike-version 2 \
   --shared-secret [SHARED_SECRET] \
   --router vpc-demo-router1 \
   --vpn-gateway vpc-demo-vpn-gw1 \
   --interface 0

//Create the second VPN tunnel in the vpc-demo network:

- gcloud compute vpn-tunnels create vpc-demo-tunnel1 \
   --peer-gcp-gateway on-prem-vpn-gw1 \
   --region us-central1 \
   --ike-version 2 \
   --shared-secret [SHARED_SECRET] \
   --router vpc-demo-router1 \
   --vpn-gateway vpc-demo-vpn-gw1 \
   --interface 1

//Create the first VPN tunnel in the on-prem network:

- gcloud compute vpn-tunnels create on-prem-tunnel0 \
   --peer-gcp-gateway vpc-demo-vpn-gw1 \
   --region us-central1 \
   --ike-version 2 \
   --shared-secret [SHARED_SECRET] \
   --router on-prem-router1 \
   --vpn-gateway on-prem-vpn-gw1 \
   --interface 0

//Create the second VPN tunnel in the on-prem network:

- gcloud compute vpn-tunnels create on-prem-tunnel1 \
   --peer-gcp-gateway vpc-demo-vpn-gw1 \
   --region us-central1 \
   --ike-version 2 \
   --shared-secret [SHARED_SECRET] \
   --router on-prem-router1 \
   --vpn-gateway on-prem-vpn-gw1 \
   --interface 1

--Create Border Gateway Protocol (BGP)--

//Create the router interface for tunnel0 in network vpc-demo:

- gcloud compute routers add-interface vpc-demo-router1 \
   --interface-name if-tunnel0-to-on-prem \
   --ip-address 169.254.0.1 \
   --mask-length 30 \
   --vpn-tunnel vpc-demo-tunnel0 \
   --region us-central1

//Create the BGP peer for tunnel0 in network vpc-demo:

- gcloud compute routers add-bgp-peer vpc-demo-router1 \
   --peer-name bgp-on-prem-tunnel0 \
   --interface if-tunnel0-to-on-prem \
   --peer-ip-address 169.254.0.2 \
   --peer-asn 65002 \
   --region us-central1
  //Create a router interface for tunnel1 in network vpc-demo:
- gcloud compute routers add-interface vpc-demo-router1 \
   --interface-name if-tunnel1-to-on-prem \
   --ip-address 169.254.1.1 \
   --mask-length 30 \
   --vpn-tunnel vpc-demo-tunnel1 \
   --region us-central1

//Create the BGP peer for tunnel1 in network vpc-demo

- gcloud compute routers add-bgp-peer vpc-demo-router1 \
   --peer-name bgp-on-prem-tunnel1 \
   --interface if-tunnel1-to-on-prem \
   --peer-ip-address 169.254.1.2 \
   --peer-asn 65002 \
   --region us-central1

//Create a router interface for tunnel0 in network on-prem:

- gcloud compute routers add-interface on-prem-router1 \
   --interface-name if-tunnel0-to-vpc-demo \
   --ip-address 169.254.0.2 \
   --mask-length 30 \
   --vpn-tunnel on-prem-tunnel0 \
   --region us-central1

//Create the BGP peer for tunnel0 in network on-prem:

- gcloud compute routers add-bgp-peer on-prem-router1 \
   --peer-name bgp-vpc-demo-tunnel0 \
   --interface if-tunnel0-to-vpc-demo \
   --peer-ip-address 169.254.0.1 \
   --peer-asn 65001 \
   --region us-central1

//Create a router interface for tunnel1 in network on-prem:

- gcloud compute routers add-interface on-prem-router1 \
   --interface-name if-tunnel1-to-vpc-demo \
   --ip-address 169.254.1.2 \
   --mask-length 30 \
   --vpn-tunnel on-prem-tunnel1 \
   --region us-central1

//Create the BGP peer for tunnel1 in network on-prem:

- gcloud compute routers add-bgp-peer on-prem-router1 \
   --peer-name bgp-vpc-demo-tunnel1 \
   --interface if-tunnel1-to-vpc-demo \
   --peer-ip-address 169.254.1.1 \
   --peer-asn 65001 \
   --region us-central1

--Verify router configurations--

//View details of Cloud Router vpc-demo-router1 to verify its settings:

- gcloud compute routers describe vpc-demo-router1 \
   --region us-central1

//View details of Cloud Router on-prem-router1 to verify its settings:

- gcloud compute routers describe on-prem-router1 \
   --region us-central1

--Configure firewall rules to allow traffic from the remote VPC--

//Allow traffic from network VPC on-prem to vpc-demo:

- gcloud compute firewall-rules create vpc-demo-allow-subnets-from-on-prem \
   --network vpc-demo \
   --allow tcp,udp,icmp \
   --source-ranges 192.168.1.0/24

--Verify the status of the tunnels--

// List the VPN tunnels you just created:

- gcloud compute vpn-tunnels list

//Verify that vpc-demo-tunnel0 tunnel is up:

- gcloud compute vpn-tunnels describe vpc-demo-tunnel0 \
   --region us-central1

//Verify that vpc-demo-tunnel1 tunnel is up:

- gcloud compute vpn-tunnels describe vpc-demo-tunnel1 \
   --region us-central1

//Verify that on-prem-tunnel0 tunnel is up:

- gcloud compute vpn-tunnels describe on-prem-tunnel0 \
   --region us-central1

//Verify that on-prem-tunnel1 tunnel is up:

- gcloud compute vpn-tunnels describe on-prem-tunnel1 \
   --region us-central1

//Open a new Cloud Shell tab and type the following to connect via SSH to the instance on-prem-instance1:

- gcloud compute ssh on-prem-instance1 --zone us-central1-a

//From the instance on-prem-instance1 in network on-prem, to reach instances in network vpc-demo, ping 10.1.1.2:

- ping -c 4 10.1.1.2

//Open a new Cloud Shell tab and update the bgp-routing mode from vpc-demo to GLOBAL:

- gcloud compute networks update vpc-demo --bgp-routing-mode GLOBAL

//Verify the change:

- gcloud compute networks describe vpc-demo

//From the Cloud Shell tab that is currently connected to the instance in network on-prem via ssh, ping the instance vpc-demo-instance2 in region us-east1:

- ping -c 2 10.2.1.2

--Verify and test the configuration of HA VPN tunnels--

//Bring tunnel0 in network vpc-demo down:

- gcloud compute vpn-tunnels delete vpc-demo-tunnel0 --region us-central1

//Verify that the tunnel is down:

- gcloud compute vpn-tunnels describe on-prem-tunnel0 --region us-central1

//Switch to the previous Cloud Shell tab that has the open ssh session running, and verify the pings between the instances in network vpc-demo and network on-prem:

- ping -c 3 10.1.1.2

--Clean up lab environment--

//From Cloud Shell, type the following commands to delete the remaining tunnels. Type "y" to confirm each action when asked:

- gcloud compute vpn-tunnels delete on-prem-tunnel0 --region us-central1

- gcloud compute vpn-tunnels delete vpc-demo-tunnel1 --region us-central1

- gcloud compute vpn-tunnels delete on-prem-tunnel1 --region us-central1

--Remove BGP peering--

//Type the following commands from each BGP peer to remove peering:

- gcloud compute routers remove-bgp-peer vpc-demo-router1 --peer-name bgp-on-prem-tunnel0 --region us-central1

- gcloud compute routers remove-bgp-peer vpc-demo-router1 --peer-name bgp-on-prem-tunnel1 --region us-central1

- gcloud compute routers remove-bgp-peer on-prem-router1 --peer-name bgp-vpc-demo-tunnel0 --region us-central1

- gcloud compute routers remove-bgp-peer on-prem-router1 --peer-name bgp-vpc-demo-tunnel1 --region us-central1

--Delete cloud routers--

//Type each command to delete the routers. Type "y" to confirm each action when asked

- gcloud compute routers delete on-prem-router1 --region us-central1

- gcloud compute routers delete vpc-demo-router1 --region us-central1

--Delete VPN gateways--

//Type each command to delete the VPN gateways. Type "y" to confirm each action when asked:

- gcloud compute vpn-gateways delete vpc-demo-vpn-gw1 --region us-central1

- gcloud compute vpn-gateways delete on-prem-vpn-gw1 --region us-central1

--Delete instances--

- gcloud compute instances delete vpc-demo-instance1 --zone us-central1-b

- gcloud compute instances delete vpc-demo-instance2 --zone us-east1-b

- gcloud compute instances delete on-prem-instance1 --zone us-central1-a

--Delete firewall rules--

//Type the following to delete the firewall rules. Type "y" to confirm each action when asked:

- gcloud compute firewall-rules delete vpc-demo-allow-custom

- gcloud compute firewall-rules delete on-prem-allow-subnets-from-vpc-demo

- gcloud compute firewall-rules delete on-prem-allow-ssh-icmp

- gcloud compute firewall-rules delete on-prem-allow-custom

- gcloud compute firewall-rules delete vpc-demo-allow-subnets-from-on-prem

- gcloud compute firewall-rules delete vpc-demo-allow-ssh-icmp

--Delete subnets--

//Type the following to delete the subnets. Type "y" to confirm each action when asked:

- gcloud compute networks subnets delete vpc-demo-subnet1 --region us-central1

- gcloud compute networks subnets delete vpc-demo-subnet2 --region us-east1

- gcloud compute networks subnets delete on-prem-subnet1 --region us-central1

--Delete VPC--

//Type these commands to delete the VPCs. Type "y" to confirm each action when asked:

- gcloud compute networks delete vpc-demo

- gcloud compute networks delete on-prem
