# Deploying an Ubuntu-based NVA in Azure with CLI

This document shows how to deploy an Ubuntu-based NVA to Azure. It include sections on basic deployment, BGP configuration with BIRD, IPsec configuration with StrongSwan and SNAT configuration with iptables. Note that all commands follow Linux shell syntax, so if you are running in Windows you might want to try them out in WSL (Windows Subsystem for Linux).

## Basic deployment

You can use the following commands in a Linux shell to deploy an Ubuntu-based NVA. The software packages BIRD (for BGP) and StrongSwan (for IPsec) will be installed:

```bash
# Variables
rg=nva
location=westeurope
vnet_name=nva
vnet_prefix=192.168.0.0/16
nva_subnet_name=nva
nva_subnet_prefix=192.168.0.0/24
nva_name=mynva
nva_pip_name="${nva_name}-pip"
nva_vm_size=Standard_B1s
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
echo "Creating RG and VNet..."
az group create -n $rg -l $location -o none
az network vnet create -n $vnet_name -g $rg --address-prefixes $vnet_prefix --subnet-name $nva_subnet_name --subnet-prefixes $nva_subnet_prefix -o none
# NSG for NVA
echo "Creating NSG ${nva_name}-nsg..."
az network nsg create -n "${nva_name}-nsg" -g $rg -o none
az network nsg rule create -n SSH --nsg-name "${nva_name}-nsg" -g $rg --priority 1000 --destination-port-ranges 22 --access Allow --protocol Tcp -o none
az network nsg rule create -n IKE --nsg-name "${nva_name}-nsg" -g $rg --priority 1010 --destination-port-ranges 4500 --access Allow --protocol Udp -o none
az network nsg rule create -n IPsec --nsg-name "${nva_name}-nsg" -g $rg --priority 1020 --destination-port-ranges 500 --access Allow --protocol Udp -o none
az network nsg rule create -n ICMP --nsg-name "${nva_name}-nsg" -g $rg --priority 1030 --destination-port-ranges '*' --access Allow --protocol Icmp -o none
# Cloudinit file for NVA
linuxnva_cloudinit_file=/tmp/linuxnva_cloudinit.txt
cat <<EOF > $linuxnva_cloudinit_file
#cloud-config
runcmd:
  - apt update && apt install -y bird strongswan
  - sysctl -w net.ipv4.ip_forward=1
  - sysctl -w net.ipv4.conf.all.accept_redirects=0 
  - sysctl -w net.ipv4.conf.all.send_redirects=0
EOF
# VM for NVA
echo "Creating VM $nva_name..."
az vm create -n $nva_name -g $rg -l $location --image Ubuntu2204 --generate-ssh-keys \
    --public-ip-address $nva_pip_name --public-ip-sku Standard --vnet-name $vnet_name --size $nva_vm_size --subnet $nva_subnet_name \
    --custom-data $linuxnva_cloudinit_file --nsg "${nva_name}-nsg" -o none
nva_nic_id=$(az vm show -n $nva_name -g "$rg" --query 'networkProfile.networkInterfaces[0].id' -o tsv)
az network nic update --ids $nva_nic_id --ip-forwarding -o none
echo "Getting information about the created VM..."
nva_pip_ip=$(az network public-ip show -n $nva_pip_name -g $rg --query ipAddress -o tsv) && echo $nva_pip_ip
nva_private_ip=$(az network nic show --ids $nva_nic_id --query 'ipConfigurations[0].privateIpAddress' -o tsv) && echo $nva_private_ip
nva_default_gw=$(first_ip "$nva_subnet_prefix") && echo $nva_default_gw
```

## BGP configuration

If all you need is BGP, you can easily configure it inline. For example, if you need to peer to Azure Route Server, you will get some parameters of ARS, and to set some variables:

```bash
# Variables
rs_name=name_of_your_azure_route_server
rs_ip1=$(az network routeserver show -n $rs_name -g $rg --query 'virtualRouterIps[0]' -o tsv) && echo $rs_ip1
rs_ip2=$(az network routeserver show -n $rs_name -g $rg --query 'virtualRouterIps[1]' -o tsv) && echo $rs_ip2
rs_asn=$(az network routeserver show -n $rs_name -g $rg --query 'virtualRouterAsn' -o tsv) && echo $rs_asn
nva_asn=65001
```

Now we can create a local file with our BIRD config, push it to the NVA, and restart the BIRD daemon

```bash
# Configure BIRD
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
      disabled;
}
protocol static {
      import all;
      route $rs_ip1/32 via $nva_default_gw;
      route $rs_ip2/32 via $nva_default_gw;
}
protocol bgp rs0 {
      description "RouteServer instance 0";
      multihop;
      local $nva_private_ip as $nva_asn;
      neighbor $rs_ip1 as $rs_asn;
          import filter {accept;};
          export filter {accept;};
}
protocol bgp rs1 {
      description "Route Server instance 1";
      multihop;
      local $nva_private_ip as $nva_asn;
      neighbor $rs_ip2 as $rs_asn;
          import filter {accept;};
          export filter {accept;};
}
EOF
username=$(whoami)
scp -P 1022 $bird_config_file "${nva_lb_ext_pip_ip}:/home/${username}/bird.conf"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_lb_ext_pip_ip "sudo mv /home/${username}/bird.conf /etc/bird/bird.conf"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_lb_ext_pip_ip "sudo systemctl restart bird"
```

## IPsec configuration

First you need to extract information from the VPN gateways in Azure. For example, if you are using Virtual WAN:

```bash
# Get VPN GW information from Virtual WAN
vwan_vpngw_name=name_of_your_vwan_vpn_gateway
vpn_psk=your_ipsec_preshared_key
vpngw_config=$(az network vpn-gateway show -n $vwan_vpngw_name -g $rg)
vpngw_gw0_pip=$(echo $vpngw_config | jq -r '.bgpSettings.bgpPeeringAddresses[0].tunnelIpAddresses[0]')
vpngw_gw1_pip=$(echo $vpngw_config | jq -r '.bgpSettings.bgpPeeringAddresses[1].tunnelIpAddresses[0]')
vpngw_gw0_bgp_ip=$(echo $vpngw_config | jq -r '.bgpSettings.bgpPeeringAddresses[0].defaultBgpIpAddresses[0]')
vpngw_gw1_bgp_ip=$(echo $vpngw_config | jq -r '.bgpSettings.bgpPeeringAddresses[1].defaultBgpIpAddresses[0]')
vpngw_bgp_asn=$(echo $vpngw_config | jq -r '.bgpSettings.asn')  # This is today always 65515
echo "Extracted info for hub vpn gateway: Gateway0 $vpngw_gw0_pip, $vpngw_gw0_bgp_ip. Gateway1 $vpngw_gw1_pip, $vpngw_gw0_bgp_ip. ASN $vpngw_bgp_asn"
```

Alternatively, if your VPN GW is non-VWAN:

```bash
# Get VPN GW information from standalone VPN Gateway (active/active)
vpngw_name=name_of_your_vwan_vpn_gateway
vpn_psk=your_ipsec_preshared_key
vpngw_bgp_asn=$(az network vnet-gateway show -n $vpngw_name -g $rg --query 'bgpSettings.asn' -o tsv) && echo $vpnwg_bgp_asn
vpngw_gw0_pip=$(az network vnet-gateway show -n $vpngw_name -g $rg --query 'bgpSettings.bgpPeeringAddresses[0].tunnelIpAddresses[0]' -o tsv) && echo $vpngw_gw0_pip
vpngw_gw0_bgp_ip=$(az network vnet-gateway show -n $vpngw_name -g $rg --query 'bgpSettings.bgpPeeringAddresses[0].defaultBgpIpAddresses[0]' -o tsv) && echo $vpngw_gw0_bgp_ip
vpngw_gw1_pip=$(az network vnet-gateway show -n $vpngw_name -g $rg --query 'bgpSettings.bgpPeeringAddresses[1].tunnelIpAddresses[0]' -o tsv) && echo $vpngw_gw1_pip
vpngw_gw1_bgp_ip=$(az network vnet-gateway show -n $vpngw_name -g $rg --query 'bgpSettings.bgpPeeringAddresses[1].defaultBgpIpAddresses[0]' -o tsv) && echo $vpngw_gw1_bgp_ip
echo "Extracted info for vpn gateway: Gateway0 $vpngw_gw0_pip, $vpngw_gw0_bgp_ip. Gateway1 $vpngw_gw1_pip, $vpngw_gw0_bgp_ip. ASN $vpngw_bgp_asn"
```

For BGP to work on IPsec, we need to create soem VTIs (Virtual Tunnel Interfaces). Note the keys (11 and 12 in this example), because we will use them later.

```bash
# VTI interfaces and static routes
# Note these changes are not reboot-persistent!!!
echo "Configuring VPN between Azure:${vpngw_gw0_pip}/${vpngw_gw0_bgp_ip} and B:${nva_pip_ip}/${nva_private_ip}..."
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip tunnel add vti0 local $nva_private_ip remote  $vpngw_gw0_pip mode vti key 12"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo sysctl -w net.ipv4.conf.vti0.disable_policy=1"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip link set up dev vti0"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip route add ${vpngw_gw0_bgp_ip}/32 dev vti0"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip tunnel add vti1 local $nva_private_ip remote  $vpngw_gw1_pip mode vti key 11"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo sysctl -w net.ipv4.conf.vti1.disable_policy=1"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip link set up dev vti1"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip route add ${vpngw_gw1_bgp_ip}/32 dev vti1"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo sed -i 's/# install_routes = yes/install_routes = no/' /etc/strongswan.d/charon.conf"
myip=$(curl -s4 ifconfig.co) && echo $myip
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip route add ${vpngw_gw0_pip}/32 via $nva_default_gw"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip route add ${vpngw_gw1_pip}/32 via $nva_default_gw"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo ip route add ${myip}/32 via $nva_default_gw" # To not lose SSH connectivity
```

The files `/etc/ipsec.secrets` and `/etc/ipsec.conf` are required to configure the IPsec tunnel. Note that compression is disabled in `ipsec.conf`, which is not compatible with VTI-based IPsec tunnels in StrongSwan:

```bash
# IPsec config files
vpn_psk_file=/tmp/ipsec.secrets
cat <<EOF > $vpn_psk_file
$nva_pip_ip  $vpngw_gw0_pip : PSK "$vpn_psk"
$nva_pip_ip  $vpngw_gw1_pip : PSK "$vpn_psk"
EOF
ipsec_file=/tmp/ipsec.conf
cat <<EOF > $ipsec_file
config setup
        charondebug="all"
        uniqueids=yes
        strictcrlpolicy=no
conn vng0
  authby=secret
  leftid=$nva_pip_ip
  leftsubnet=0.0.0.0/0
  right= $vpngw_gw0_pip
  rightsubnet=0.0.0.0/0
  keyexchange=ikev2
  ikelifetime=28800s
  keylife=3600s
  keyingtries=3
  compress=no
  auto=start
  ike=aes256-sha1-modp1024
  esp=aes256-sha1
  mark=12
conn vng1
  authby=secret
  leftid=$nva_pip_ip
  leftsubnet=0.0.0.0/0
  right= $vpngw_gw1_pip
  rightsubnet=0.0.0.0/0
  keyexchange=ikev2
  ikelifetime=28800s
  keylife=3600s
  keyingtries=3
  compress=yes
  auto=start
  ike=aes256-sha1-modp1024
  esp=aes256-sha1
  mark=11
EOF
# Copy files to NVA and restart ipsec daemon
username=$(whoami)
scp $vpn_psk_file $nva_pip_ip:/home/$username/ipsec.secrets
scp $ipsec_file $nva_pip_ip:/home/$username/ipsec.conf
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo mv ./ipsec.* /etc/"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo systemctl restart ipsec"
```

```bash
# Configure BGP with Bird (NVA to VPNGW)
nva_asn=65001
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
      route $vnet_prefix via $nva_default_gw;
}
protocol bgp vpngw0 {
      description "VPN Gateway instance 0";
      multihop;
      local $nva_private_ip as $nva_asn;
      neighbor $vpngw_gw0_bgp_ip as $vpngw_asn;
          import filter {accept;};
          export filter {accept;};
}
protocol bgp vpngw1 {
      description "VPN Gateway instance 1";
      multihop;
      local $nva_private_ip as $nva_asn;
      neighbor $vpngw_gw1_bgp_ip as $vpngw_asn;
          import filter {accept;};
          export filter {accept;};
}
EOF
username=$(whoami)
scp $bird_config_file "${nva_pip_ip}:/home/${username}/bird.conf"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo mv /home/${username}/bird.conf /etc/bird/bird.conf"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo systemctl restart bird"
```

## Source NAT

Ubuntu comes with iptables, that can be easily configure to SNAT (aka "masquerade") IP addresses. This is especially useful if you are going to send Internet traffic from other VMs through your NVA. Here some examples to enable SNAT:

```bash
# Disable SNAT for all traffic going through the NVA
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva1_pip "sudo iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE"
```

```bash
# Disable SNAT for all IP addresses EXCEPT for the 10.0.0.0/8 range
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva1_pip "sudo iptables -t nat -D POSTROUTING ! -d '10.0.0.0/8' -o eth0 -j MASQUERADE"
```

To disable SNAT you just need to use the `-D` iptables command:

## Diagnostics and troubleshooting

Interfaces and routes:

```bash
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "sysctl -w net.ipv4.ip_forward" # Check status of IP forwarding at the OS level
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "ip a"                          # Interface status
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "ip route"                      # IP routes
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "ifconfig vti0"                 # VTI0 counters
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "ip -s tunnel show"             # Tunnel information
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "cat /proc/net/xfrm_stat"       # Advanced counters
```

Resetting VTI0:

```bash
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "ip tunnel del vti0"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "sudo ip tunnel add vti0 local 192.168.0.2 remote 40.122.240.210 mode vti key 12"
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
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "sudo ip xfrm state"            # Transform state
```

For BGP

```bash
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no -p 1022 $nva_pip_ip "systemctl status bird"         # Check status of BIRD daemon
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo birdc show status"                # BIRD status
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo birdc show protocols"             # BGP neighbors
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo birdc show protocols rs0"         # Information about a specific BGP neighbor
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo birdc show protocol all rs0"      # Additional information about a specific BGP neighbor
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo birdc show route"                 # Learned routes
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo birdc show route protocol rs0"    # Learned routes from a specific neighbor
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "sudo birdc show route export rs0"      # Advertised routes
```
