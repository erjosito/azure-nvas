# Deploying an Ubuntu-based NVA in Azure with CLI

This document shows how to deploy an Ubuntu-based NVA to Azure. It include sections on basic deployment, BGP configuration with BIRD, IPsec configuration with StrongSwan and SNAT configuration with iptables. Note that all commands follow Linux shell syntax, so if you are running in Windows you might want to try them out in WSL (Windows Subsystem for Linux).

## Basic deployment

You can use the following commands in a Linux shell to deploy an Ubuntu-based NVA. The software packages BIRD (for BGP) and StrongSwan (for IPsec) will be installed:

```bash
# Variables
rg=nva
location=eastus2
vnet_name=nva
vnet_prefix=10.13.76.0/24
nva_subnet_name=nva
nva_subnet_prefix=10.13.76.0/26
nva_name=mynva
nva_pip_name="${nva_name}-pip"
nva_vm_size=Standard_B1s
nva_asn=65001
vpn_psk=your_ipsec_preshared_key
# Function to get the first IP (default gateway) of a subnet. Example: first_ip 192.168.0.64/27
function first_ip(){
    subnet=$1
    IP=$(echo $subnet | cut -d/ -f 1)
    IP_HEX=$(printf '%.2X%.2X%.2X%.2X\n' `echo $IP | sed -e 's/\./ /g'`)
    NEXT_IP_HEX=$(printf %.8X `echo $(( 0x$IP_HEX + 1 ))`)
    NEXT_IP=$(printf '%d.%d.%d.%d\n' `echo $NEXT_IP_HEX | sed -r 's/(..)/0x\1 /g'`)
    echo "$NEXT_IP"
}
# Create RG and VNet
echo "Creating RG and VNet ${vnet_name} in ${location}..."
az group create -n $rg -l $location -o none
az network vnet create -n $vnet_name -g $rg --address-prefixes $vnet_prefix --subnet-name $nva_subnet_name --subnet-prefixes $nva_subnet_prefix -l $location -o none
# NSG for NVA
nsg_name="${nva_name}-nsg"
echo "Creating NSG $nsg_name..."
az network nsg create -n "$nsg_name" -g $rg -l $location -o none
az network nsg rule create -n SSH --nsg-name "$nsg_name" -g $rg --priority 1000 --destination-port-ranges 22 --access Allow --protocol Tcp -o none
az network nsg rule create -n IKE --nsg-name "$nsg_name" -g $rg --priority 1010 --destination-port-ranges 4500 --access Allow --protocol Udp -o none
az network nsg rule create -n IPsec --nsg-name "$nsg_name" -g $rg --priority 1020 --destination-port-ranges 500 --access Allow --protocol Udp -o none
az network nsg rule create -n ESP --nsg-name "$nsg_name" -g $rg --priority 1030 --destination-port-ranges '*' --access Allow --protocol Esp -o none
az network nsg rule create -n ICMP --nsg-name "$nsg_name" -g $rg --priority 1040 --destination-port-ranges '*' --access Allow --protocol Icmp -o none
# Cloudinit file for NVA
linuxnva_cloudinit_file=/tmp/linuxnva_cloudinit.txt
cat <<EOF > $linuxnva_cloudinit_file
#cloud-config
runcmd:
  - apt update && apt install -y bird strongswan strongswan-swanctl
  - sysctl -w net.ipv4.ip_forward=1
  - sysctl -w net.ipv4.conf.all.accept_redirects=0 
  - sysctl -w net.ipv4.conf.all.send_redirects=0
EOF
# VM for NVA
publisher=Canonical
offer=ubuntu-24_04-lts
sku=server
version=latest
urn="$publisher:$offer:$sku:$version"
echo "Creating VM $nva_name..."
az vm create -n $nva_name -g $rg -l $location --image $urn --generate-ssh-keys \
    --public-ip-address $nva_pip_name --public-ip-sku Standard --vnet-name $vnet_name --size $nva_vm_size --subnet $nva_subnet_name \
    --custom-data $linuxnva_cloudinit_file --nsg "${nva_name}-nsg" -o none
nva_nic_id=$(az vm show -n $nva_name -g "$rg" --query 'networkProfile.networkInterfaces[0].id' -o tsv)
az network nic update --ids $nva_nic_id --ip-forwarding -o none
echo "Getting information about the created VM..."
nva_pip_ip=$(az network public-ip show -n $nva_pip_name -g $rg --query ipAddress -o tsv) && echo "Public IP: ${nva_pip_ip}"
nva_private_ip=$(az network nic show --ids $nva_nic_id --query 'ipConfigurations[0].privateIPAddress' -o tsv) && echo "Private IP: ${nva_private_ip}"
nva_default_gw=$(first_ip "$nva_subnet_prefix") && echo "Default gateway: ${nva_default_gw}"
```

## Create Azure VPN termination device

You have two options, either a standalone VPN gateway or Virtual WAN.

### Option 1. Create Azure standalone gateway

The first option is creating an Azure VPN gateway in a VNet:

```bash
# Variables
vng_vnet_name=vng-vnet
vng_vnet_prefix=192.168.0.0/23
vng_subnet_prefix=192.168.0.0/27
vpngw_name=vpngw
lng_name=onpremnva
cx_name=onprem
# Create gateway
echo "Creating Virtual Network Gateway..."
az network vnet create -g $rg -n $vng_vnet_name --address-prefixes $vng_vnet_prefix --subnet-name GatewaySubnet --subnet-prefixes $vng_subnet_prefix -o none
az network public-ip create -g $rg -n "${vpngw_name}-pip-a" --sku standard --allocation-method static -l $location -o none
az network public-ip create -g $rg -n "${vpngw_name}-pip-b" --sku standard --allocation-method static -l $location -o none
az network vnet-gateway create --name $vpngw_name -g $rg --vnet $vng_vnet_name --public-ip-addresses "${vpngw_name}-pip-a" "${vpngw_name}-pip-b" --sku VpnGw1 --asn 65515 -o none
# Create LNG and connection
echo "Creating Local Network Gateway..."
az network local-gateway create -g $rg -n $lng_name --gateway-ip-address $nva_pip_ip --asn $nva_asn --bgp-peering-address 10.13.77.4 -o none
az network vpn-connection create -g $rg --shared-key $vpn_psk -n $cx_name --vnet-gateway1 $vpngw_name --local-gateway2 $lng_name -l $location --enable-bgp -o none
```

And now we get the information we need from the gateway:

```bash
# Get VPN GW information from standalone VPN Gateway (active/active)
vpngw_bgp_asn=$(az network vnet-gateway show -n $vpngw_name -g $rg --query 'bgpSettings.asn' -o tsv) && echo $vpnwg_bgp_asn
vpngw_gw0_pip=$(az network vnet-gateway show -n $vpngw_name -g $rg --query 'bgpSettings.bgpPeeringAddresses[0].tunnelIpAddresses[0]' -o tsv) && echo $vpngw_gw0_pip
vpngw_gw0_bgp_ip=$(az network vnet-gateway show -n $vpngw_name -g $rg --query 'bgpSettings.bgpPeeringAddresses[0].defaultBgpIpAddresses[0]' -o tsv) && echo $vpngw_gw0_bgp_ip
vpngw_gw1_pip=$(az network vnet-gateway show -n $vpngw_name -g $rg --query 'bgpSettings.bgpPeeringAddresses[1].tunnelIpAddresses[0]' -o tsv) && echo $vpngw_gw1_pip
vpngw_gw1_bgp_ip=$(az network vnet-gateway show -n $vpngw_name -g $rg --query 'bgpSettings.bgpPeeringAddresses[1].defaultBgpIpAddresses[0]' -o tsv) && echo $vpngw_gw1_bgp_ip
echo "Extracted info for vpn gateway: Gateway0 $vpngw_gw0_pip, $vpngw_gw0_bgp_ip. Gateway1 $vpngw_gw1_pip, $vpngw_gw1_bgp_ip. ASN $vpngw_bgp_asn"
```

### Option 2. Create Azure Virtual WAN

Alternatively you can create a Virtual WAN hub with a VPN gateway:

```bash
# Variables
vwan_name=vwan
hub_name=hub1
hub_prefix=192.168.0.0/23
vwan_vpngw_name=hubvpn1
# Create VWAN and hub
echo "Creating Virtual WAN..."
az network vwan create -n $vwan_name -g $rg -l $location --branch-to-branch-traffic true --type Standard -o none
az network vhub create -n $hub_name -g $rg --vwan $vwan_name -l $location --address-prefix $hub_prefix -o none
az network vpn-gateway create -n $vwan_vpngw_name -g $rg -l $location --vhub $hub_name --asn 65515 -o none
# Create site and connection
echo "Creating VPN site..."
az network vpn-site create -n $nva_name -g $rg -l $location --virtual-wan $vwan_name --device-vendor Canonical --device-model Ubuntu2404 --link-speed 100 \
      --ip-address $nva_pip_ip --asn $nva_asn --bgp-peering-address $nva_private_ip --with-link -o none
site_link_id=$(az network vpn-site link list --site-name $nva_name -g $rg --query '[0].id' -o tsv)
hub_default_rt_id=$(az network vhub route-table show --vhub-name $hub_name -g $rg -n defaultRouteTable --query id -o tsv)
az network vpn-gateway connection create -n $nva_name --gateway-name $vwan_vpngw_name -g $rg --remote-vpn-site $nva_name --with-link true --vpn-site-link $site_link_id \
      --enable-bgp true --protocol-type IKEv2 --shared-key "$vpn_psk" --connection-bandwidth 100 --routing-weight 10 \
      --associated-route-table $hub_default_rt_id --propagated-route-tables $hub_default_rt_id --labels default --internet-security true -o none
```

And we need to extract information from the VPN gateways in the virtual hub:

```bash
# Get VPN GW information from Virtual WAN
vpngw_config=$(az network vpn-gateway show -n $vwan_vpngw_name -g $rg)
vpngw_gw0_pip=$(echo $vpngw_config | jq -r '.bgpSettings.bgpPeeringAddresses[0].tunnelIpAddresses[0]')
vpngw_gw1_pip=$(echo $vpngw_config | jq -r '.bgpSettings.bgpPeeringAddresses[1].tunnelIpAddresses[0]')
vpngw_gw0_bgp_ip=$(echo $vpngw_config | jq -r '.bgpSettings.bgpPeeringAddresses[0].defaultBgpIpAddresses[0]')
vpngw_gw1_bgp_ip=$(echo $vpngw_config | jq -r '.bgpSettings.bgpPeeringAddresses[1].defaultBgpIpAddresses[0]')
vpngw_bgp_asn=$(echo $vpngw_config | jq -r '.bgpSettings.asn')  # This is today always 65515
echo "Extracted info for hub vpn gateway: Gateway0 $vpngw_gw0_pip, $vpngw_gw0_bgp_ip. Gateway1 $vpngw_gw1_pip, $vpngw_gw1_bgp_ip. ASN $vpngw_bgp_asn"
```

## IPsec+BGP configuration with XFRM interfaces

This guide is following the docs in [here](https://docs.strongswan.org/docs/5.9/features/routeBasedVpn.html#_xfrm_interface_management). XFRM are a new type of interfaces that replace VTI. Make sure you have the right variables from previous steps and that BIRD and StrongSwan are installed:

```bash
# Function to get the first IP (default gateway) of a subnet. Example: first_ip 192.168.0.64/27
function first_ip(){
    subnet=$1
    IP=$(echo $subnet | cut -d/ -f 1)
    IP_HEX=$(printf '%.2X%.2X%.2X%.2X\n' `echo $IP | sed -e 's/\./ /g'`)
    NEXT_IP_HEX=$(printf %.8X `echo $(( 0x$IP_HEX + 1 ))`)
    NEXT_IP=$(printf '%d.%d.%d.%d\n' `echo $NEXT_IP_HEX | sed -r 's/(..)/0x\1 /g'`)
    echo "$NEXT_IP"
}
# VM variables
echo "Getting information about the created VM ${nva_name}..."
nva_nic_id=$(az vm show -n $nva_name -g "$rg" --query 'networkProfile.networkInterfaces[0].id' -o tsv)
nva_pip_id=$(az network nic show --ids $nva_nic_id --query 'ipConfigurations[0].publicIPAddress.id' -o tsv)
nva_pip_ip=$(az network public-ip show --ids $nva_pip_id --query ipAddress -o tsv) && echo "Public IP address: ${nva_pip_ip}"
nva_private_ip=$(az network nic show --ids $nva_nic_id --query 'ipConfigurations[0].privateIPAddress' -o tsv) && echo "Private IP address: ${nva_private_ip}"
nva_subnet_id=$(az network nic show --ids $nva_nic_id --query 'ipConfigurations[0].subnet.id' -o tsv)
nva_subnet_prefix=$(az network vnet subnet show --ids $nva_subnet_id --query addressPrefix -o tsv)
nva_default_gw=$(first_ip "$nva_subnet_prefix") && echo "Default gateway: ${nva_default_gw}"
# This step verifies that you have SSH connectivity, and that you have BIRD and StrongSwan installed
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo apt update && sudo apt install -y bird strongswan strongswan-swanctl"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo sysctl -w net.ipv4.ip_forward=1"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo sysctl -w net.ipv4.conf.all.accept_redirects=0"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo sysctl -w net.ipv4.conf.all.send_redirects=0"
```

First of all, you might want to create some new user with password, in case you need to access the VM over the console (some times I have lost connectivity to the VM):

```bash
ssh $nva_pip_ip
sudo adduser consoleadmin
sudo usermod -aG sudo consoleadmin
```

You could enable boot diagnostics, now that you are at it:

```bash
az vm boot-diagnostics enable -n $nva_name -g $rg -o none
```

Instead of the VTI interfaces, XRFM interfaces are created. No marks or endpoints are required, but an interface ID is assigned (`41` and `42` in this example):

```bash
# XRFM interfaces and static routes
# Note these changes are not reboot-persistent!!!
echo "Creating XFRM interfaces..."
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip link add ipsec0 type xfrm dev eth0 if_id 41"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip link set ipsec0 up"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip route add ${vpngw_gw0_bgp_ip}/32 dev ipsec0"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip link add ipsec1 type xfrm dev eth0 if_id 42"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip link set ipsec1 up"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip route add ${vpngw_gw1_bgp_ip}/32 dev ipsec1"
echo "Adding routes..."
# Disabling route installation is not required any more
# ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo sed -i 's/# install_routes = yes/install_routes = no/' /etc/strongswan.d/charon.conf"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip route add ${vpngw_gw0_pip}/32 via $nva_default_gw"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip route add ${vpngw_gw1_pip}/32 via $nva_default_gw"
myip=$(curl -s4 ifconfig.co) && echo "Installing route for $myip..."
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip route add ${myip}/32 via $nva_default_gw" # To not lose SSH connectivity
# However, we need to install throw routes in route table 220
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip route add throw ${vpngw_gw0_pip}/32 table 220"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip route add throw ${vpngw_gw1_pip}/32 table 220"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip route add throw ${myip}/32 table 220"
# Check interfaces and routes
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "ip a & ip route"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip xfrm policy"
```

The new configuration file is `/etc/swanctl/swanctl.conf`. The config is then loaded with `swanctl --load-all`:

```bash
# StrongSwan config file
swanctl_file=/tmp/swanctl.conf
cat <<EOF > $swanctl_file
connections {
   vng0 {
        local_addrs  = $nva_private_ip
        remote_addrs = $vpngw_gw0_pip
        version = 2
        proposals = aes256-sha1-modp1024,aes192-sha256-modp3072,default
        keyingtries = 0
        encap = yes
        local {
            auth = psk
            id = $nva_pip_ip
        }
        remote {
            auth = psk
            id = $vpngw_gw0_pip
            revocation = relaxed
        }
        children {
            s2s0 {
                local_ts = 0.0.0.0/0
                remote_ts = 0.0.0.0/0
                esp_proposals = aes256-sha1,default
                dpd_action = restart
                start_action = trap
                rekey_time = 3600
            }
        }
        if_id_in = 41
        if_id_out = 41
   }
   vng1 {
        local_addrs  = $nva_private_ip
        remote_addrs = $vpngw_gw1_pip
        version = 2
        proposals = aes256-sha1-modp1024,aes192-sha256-modp3072,default
        keyingtries = 0
        encap = yes
        local {
            auth = psk
            id = $nva_pip_ip
        }
        remote {
            auth = psk
            id = $vpngw_gw1_pip
            revocation = relaxed
        }
        children {
            s2s1 {
                local_ts = 0.0.0.0/0
                remote_ts = 0.0.0.0/0
                esp_proposals = aes256-sha1,default
                dpd_action = restart
                start_action = trap
                rekey_time = 3600
            }
        }
        if_id_in = 42
        if_id_out = 42
   }
}
secrets {
   # PSK secret
   ike-1 {
        id-0 = $vpngw_gw0_pip
        id-1 = $vpngw_gw1_pip
        secret = "$vpn_psk" 
   }
}
EOF
# Copy files to NVA and restart ipsec daemon
username=$(whoami)
scp $swanctl_file $nva_pip_ip:/home/$username/swanctl.conf
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo mv ./swanctl.conf /etc/swanctl/"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo systemctl restart ipsec"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo swanctl --load-all"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ipsec status"
```

For the BGP part, StrongSwan on XFRM interfaces works on route table 220, not sure if the kernel protocol needs to be associated to that RT.

```bash
# Configure BGP with Bird (NVA to VPNGW)
bird_config_file=/tmp/bird.conf
cat <<EOF > $bird_config_file
log syslog all;
router id $nva_private_ip;
protocol device {
        scan time 10;
}
protocol direct {
      disabled;
}
protocol kernel {
      preference 254;
      learn;
      merge paths on;
      import filter {
          if net ~ ${vpngw_gw0_bgp_ip}/32 then accept;
          if net ~ ${vpngw_gw1_bgp_ip}/32 then accept;
          else reject;
      };
      export filter {
          if net ~ ${vpngw_gw0_bgp_ip}/32 then reject;
          else accept;
      };
}
protocol static {
      import all;
      # Test route
      route 2.2.2.2/32 via $nva_default_gw;
      route $nva_subnet_prefix via $nva_default_gw;
}
protocol bgp vpngw0 {
      description "VPN Gateway instance 0";
      multihop;
      local $nva_private_ip as $nva_asn;
      neighbor $vpngw_gw0_bgp_ip as $vpngw_bgp_asn;
          import filter {accept;};
          export filter {accept;};
}
protocol bgp vpngw1 {
      description "VPN Gateway instance 1";
      multihop;
      local $nva_private_ip as $nva_asn;
      neighbor $vpngw_gw1_bgp_ip as $vpngw_bgp_asn;
          import filter {accept;};
          export filter {accept;};
}
EOF
username=$(whoami)
scp $bird_config_file "${nva_pip_ip}:/home/${username}/bird.conf"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo mv /home/${username}/bird.conf /etc/bird/bird.conf"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo systemctl restart bird"
```

## Diagnostics and troubleshooting

Interfaces and routes:

```bash
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "sysctl -w net.ipv4.ip_forward" # Check status of IP forwarding at the OS level
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "ip a"                          # Interface status
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "ip route"                      # IP route
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "ip route show table all"       # IP routes (all tables, including 220)
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "ifconfig ipsec0"               # ipsec0 counters
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "ip -s tunnel show"             # Tunnel information
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "cat /proc/net/xfrm_stat"       # Advanced counters
```

iptables:

```bash
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "sudo iptables -L"              # Show iptables rules
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "sudo iptables -t nat -L"       # Show iptables NAT rules
```

For IPsec:

```bash
# Diagnostic commands for Ubuntu-based NVA
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "systemctl status ipsec"        # Check status of StrongSwan daemon
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ipsec status"                     # IPsec status
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ipsec statusall"                  # IPsec status
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo swanctl -l"                       # Swanctl
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo swanctl -L"                       # Swanctl
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "sudo ip xfrm state"            # Transform state
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "sudo ip xfrm policy"           # Transform policy
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "sudo swanctl --initiate --ike vng0 --child s2s0"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "sudo swanctl --initiate --ike vng1 --child s2s1"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "sudo swanctl --list-conns"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "sudo swanctl --list-sas"
```

For BGP and BIRD:

```bash
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "systemctl status bird"            # Check status of BIRD daemon
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo birdc show status"                   # BIRD status
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo birdc show protocols"                # BGP neighbors
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo birdc show protocols vpngw0"         # Information about a specific BGP neighbor
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo birdc show protocol all vpngw0"      # Additional information about a specific BGP neighbor
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo birdc show route"                    # Learned routes
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo birdc show route protocol vpngw0"    # Learned routes from a specific neighbor
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo birdc show route export vpngw0"      # Advertised routes
```

## Troubleshooting examples

### Example 1: wrong BGP IP configured in the Azure Local Network Gateway

The BIRD adjacencies might stay down, showing a message of `No route to host`:

```
jose@mynva:~$ sudo birdc show prot
BIRD 1.6.8 ready.
name     proto    table    state  since       info
device1  Device   master   up     11:56:19
direct1  Direct   master   down   11:56:19
kernel1  Kernel   master   up     11:56:19
static1  Static   master   up     11:56:19
vpngw0   BGP      master   start  11:56:19    Connect       Socket: No route to host
vpngw1   BGP      master   start  11:56:19    Connect       Socket: No route to host
```

You should verify that the IPsec connections are up and running:

```
jose@mynva:~$ sudo ipsec status
Routed Connections:
        s2s1{2}:  ROUTED, TUNNEL, reqid 2
        s2s1{2}:   0.0.0.0/0 === 0.0.0.0/0
        s2s0{1}:  ROUTED, TUNNEL, reqid 1
        s2s0{1}:   0.0.0.0/0 === 0.0.0.0/0
Security Associations (2 up, 0 connecting):
        vng0[18]: ESTABLISHED 2 hours ago, 10.13.76.4[52.251.117.90]...20.15.113.219[20.15.113.219]
        s2s0{11}:  INSTALLED, TUNNEL, reqid 1, ESP in UDP SPIs: cace17a2_i 830e157c_o
        s2s0{11}:   0.0.0.0/0 === 0.0.0.0/0
        vng1[17]: ESTABLISHED 2 hours ago, 10.13.76.4[52.251.117.90]...172.172.62.91[172.172.62.91]
        s2s1{12}:  INSTALLED, TUNNEL, reqid 3, ESP in UDP SPIs: c7db29a2_i ee7fa93e_o
        s2s1{12}:   0.0.0.0/0 === 0.0.0.0/0
        s2s1{13}:  INSTALLED, TUNNEL, reqid 2, ESP in UDP SPIs: cc9d0267_i b8dd57de_o
        s2s1{13}:   0.0.0.0/0 === 0.0.0.0/0
```

If you inspect the detailed command `ipsec statusall` you can get the packet counters. In this case, you can observe that there are zero bytes of outbound traffic for one of the security associations, which might be part of the problem:

```
        s2s0{1}:   0.0.0.0/0 === 0.0.0.0/0
Security Associations (2 up, 0 connecting):
        vng0[18]: ESTABLISHED 2 hours ago, 10.13.76.4[52.251.117.90]...20.15.113.219[20.15.113.219]
        vng0[18]: IKEv2 SPIs: 89215c3ec9ce6006_i* 083b45149810a93d_r, rekeying in 66 minutes
        vng0[18]: IKE proposal: AES_CBC_256/HMAC_SHA1_96/PRF_HMAC_SHA1/MODP_1024
        s2s0{14}:  INSTALLED, TUNNEL, reqid 1, ESP in UDP SPIs: ca8c0aaf_i 1bd1e14a_o
        s2s0{14}:  AES_GCM_16_256, 3172 bytes_i (61 pkts, 1s ago), 840 bytes_o (14 pkts, 20s ago), rekeying in 55 minutes
        s2s0{14}:   0.0.0.0/0 === 0.0.0.0/0
        vng1[17]: ESTABLISHED 2 hours ago, 10.13.76.4[52.251.117.90]...172.172.62.91[172.172.62.91]
        vng1[17]: IKEv2 SPIs: 7d330f1835b8b6b6_i 892a3a5459d76c75_r*, rekeying in 57 minutes
        vng1[17]: IKE proposal: AES_CBC_256/HMAC_SHA1_96/PRF_HMAC_SHA1/MODP_1024
        s2s1{12}:  INSTALLED, TUNNEL, reqid 3, ESP in UDP SPIs: c7db29a2_i ee7fa93e_o
        s2s1{12}:  AES_GCM_16_256, 312 bytes_i (6 pkts, 427s ago), 0 bytes_o, rekeying in 48 minutes
        s2s1{12}:   0.0.0.0/0 === 0.0.0.0/0
        s2s1{13}:  INSTALLED, TUNNEL, reqid 2, ESP in UDP SPIs: cc9d0267_i b8dd57de_o
        s2s1{13}:  AES_GCM_16_256, 9464 bytes_i (182 pkts, 0s ago), 2640 bytes_o (44 pkts, 11s ago), rekeying in 47 minutes
        s2s1{13}:   0.0.0.0/0 === 0.0.0.0/0
```

You can try to ping the remote gateways, the BGP addresses should answer to ICMP echo requests. In this case, we are getting no answer:

```
jose@mynva:~$ ping 192.168.0.4
PING 192.168.0.4 (192.168.0.4) 56(84) bytes of data.
^C
--- 192.168.0.4 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 4124ms

jose@mynva:~$ ping 192.168.0.5
PING 192.168.0.5 (192.168.0.5) 56(84) bytes of data.
^C
--- 192.168.0.5 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3093ms
```

You can check whether you are getting TCP packets from the gateway on the XFRM interfaces, and whether you are sending them on the correct interfaces. As you can see in the following output, the VPN gateway is sending packets to the wrong IP address `10.13.77.4`, while our IP NVA has the IP address of `10.13.76.4` (as you can see in the locally originated packets):

```
jose@mynva:~$ sudo tcpdump -i any -n port 179
tcpdump: data link type LINUX_SLL2
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
14:58:39.727322 ipsec1 In  IP 192.168.0.5.50997 > 10.13.77.4.179: Flags [SEW], seq 4012300367, win 64240, options [mss 1360,nop,wscale 8,nop,nop,sackOK], length 0
14:58:39.727336 eth0  Out IP 192.168.0.5.50997 > 10.13.77.4.179: Flags [SEW], seq 4012300367, win 64240, options [mss 1360,nop,wscale 8,nop,nop,sackOK], length 0
14:58:40.143585 ipsec0 In  IP 192.168.0.4.59552 > 10.13.77.4.179: Flags [SEW], seq 3326914158, win 64240, options [mss 1360,nop,wscale 8,nop,nop,sackOK], length 0
14:58:40.143603 eth0  Out IP 192.168.0.4.59552 > 10.13.77.4.179: Flags [SEW], seq 3326914158, win 64240, options [mss 1360,nop,wscale 8,nop,nop,sackOK], length 0
14:58:40.583554 ipsec0 Out IP 10.13.76.4.54069 > 192.168.0.4.179: Flags [S], seq 323564816, win 32120, options [mss 1460,sackOK,TS val 2462243334 ecr 0,nop,wscale 7], length 0
14:58:41.734581 ipsec1 In  IP 192.168.0.5.50997 > 10.13.77.4.179: Flags [S], seq 4012300367, win 64240, options [mss 1360,nop,wscale 8,nop,nop,sackOK], length 0
14:58:41.734600 eth0  Out IP 192.168.0.5.50997 > 10.13.77.4.179: Flags [S], seq 4012300367, win 64240, options [mss 1360,nop,wscale 8,nop,nop,sackOK], length 0
14:58:42.534152 ipsec1 In  IP 192.168.0.5.50999 > 10.13.77.4.179: Flags [SEW], seq 3850790282, win 64240, options [mss 1360,nop,wscale 8,nop,nop,sackOK], length 0
14:58:42.534177 eth0  Out IP 192.168.0.5.50999 > 10.13.77.4.179: Flags [SEW], seq 3850790282, win 64240, options [mss 1360,nop,wscale 8,nop,nop,sackOK], length 0
14:58:45.751857 ipsec0 In  IP 192.168.0.4.59553 > 10.13.77.4.179: Flags [SEW], seq 3271099428, win 64240, options [mss 1360,nop,wscale 8,nop,nop,sackOK], length 0
14:58:45.751877 eth0  Out IP 192.168.0.4.59553 > 10.13.77.4.179: Flags [SEW], seq 3271099428, win 64240, options [mss 1360,nop,wscale 8,nop,nop,sackOK], length 0
14:58:46.758200 ipsec0 In  IP 192.168.0.4.59553 > 10.13.77.4.179: Flags [SEW], seq 3271099428, win 64240, options [mss 1360,nop,wscale 8,nop,nop,sackOK], length 0
14:58:46.758217 eth0  Out IP 192.168.0.4.59553 > 10.13.77.4.179: Flags [SEW], seq 3271099428, win 64240, options [mss 1360,nop,wscale 8,nop,nop,sackOK], length 0
```

Resolution: the Local Network Gateway in Azure had the wrong IP address configured.
