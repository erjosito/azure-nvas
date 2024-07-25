> **WARNING**: The Cisco CSR is not available anymore in Azure, but only its replacement the Cisco Catalyst 8000v. This new device has a slightly different software, so this guide will not be applicable any more.

# Deploying Cisco CSR in Azure with CLI

This document includes information about how to deploy a basic CSR configuration to Azure, as well as configuration snippets for BGP, IPSec and some useful troubleshooting commands. Note that all commands follow Linux shell syntax, so if you are running in Windows you might want to try them out in WSL (Windows Subsystem for Linux).

## Basic deployment

First we will need some variables and functions, as well as a resource group, a VNet and an NSG:

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
nva_nsg_name="${nva_name}-nsg"
nva_vm_size=Standard_B2ms
nva_username=$(whoami)
nva_password="your_super_secret_password"
publisher=cisco
offer=cisco-csr-1000v
sku=16_12-byol
version=$(az vm image list -p $publisher -f $offer -s $sku --all --query '[0].version' -o tsv)
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
# NSG for onprem NVA
echo "Creating NSG ${nva_nsg_name}..."
az network nsg create -n "${nva_nsg_name}" -g $rg -o none
az network nsg rule create -n SSH --nsg-name "${nva_nsg_name}" -g $rg --priority 1000 --destination-port-ranges 22 --access Allow --protocol Tcp -o none
az network nsg rule create -n IKE --nsg-name "${nva_nsg_name}" -g $rg --priority 1010 --destination-port-ranges 4500 --access Allow --protocol Udp -o none
az network nsg rule create -n IPsec --nsg-name "${nva_nsg_name}" -g $rg --priority 1020 --destination-port-ranges 500 --access Allow --protocol Udp -o none
az network nsg rule create -n ICMP --nsg-name "${nva_nsg_name}" -g $rg --priority 1030 --destination-port-ranges '*' --access Allow --protocol Icmp -o none
```

Now we can create a single-NIC CSR VM:

```bash
# Create single-NIC CSR
az network public-ip create -g $rg -n "${nva_pip_name}" --sku Standard --allocation-method Static
az network nic create -n "${nva_name}-nic0" -g $rg --vnet-name $vnet_name --subnet $nva_subnet_name --public-ip-address "${nva_pip_name}" --network-security-group "${nva_nsg_name}" --ip-forwarding
az vm image terms accept --urn ${publisher}:${offer}:${sku}:${version}
az vm create -n $nva_name -g $rg -l $location --size $nva_vm_size \
    --image ${publisher}:${offer}:${sku}:${version} \
    --admin-username "$nva_username" --admin-password $nva_password --authentication-type all --generate-ssh-keys \
    --nics "${nva_name}-nic0"
nva_private_ip=$(az network nic show -n "${nva_name}-nic0" -g $rg --query 'ipConfigurations[0].privateIpAddress' -o tsv) && echo $nva_private_ip
nva_pip_ip=$(az network public-ip show -n $nva_pip_name -g $rg --query ipAddress -o tsv) && echo $nva_pip_ip
nva_default_gw=$(first_ip $nva_subnet_prefix)
```

If you are going to need NAT, having a 2-NIC NVA might come handy:

```bash
# Create dual-NIC CSR
nva_subnet2_name=nvainternal
nva_subnet2_prefix=192.168.1.0/24
az network vnet subnet create -n $nva_subnet2_name --address-prefix $nva_subnet2_prefix --vnet-name $vnet_name -g $rg
az network public-ip create -g $rg -n "${nva_pip_name}" --sku basic --allocation-method Static
az network nic create -n "${nva_name}-nic0" -g $rg --vnet-name $vnet_name --subnet $nva_subnet1_name --network-security-group "$nva_nsg_name" --public-ip-address "${nva_pip_name}" --ip-forwarding
az network nic create -n "${nva_name}-nic1" -g $rg --vnet-name $vnet_name --subnet $hub_csrint_subnet_name --network-security-group "$nva_nsg_name" --ip-forwarding
az vm create -n $nva_name -g $rg -l $location \
    --image ${publisher}:${offer}:${sku}:${version} --size $nva_vm_size \
    --admin-username "$nva_username" --admin-password $nva_password --authentication-type all --generate-ssh-keys \
    --nics "${nva_name}-nic0" "${nva_name}-nic1"
nva_private_ip=$(az network nic show -n "${nva_name}-nic1" -g $rg --query 'ipConfigurations[0].privateIpAddress' -o tsv) && echo $nva_private_ip
nva_pip_ip=$(az network public-ip show -n $nva_pip_name -g $rg --query ipAddress -o tsv) && echo $nva_pip_ip
nva_default_gw=$(first_ip $nva_subnet2_prefix)
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

You can now configure BGP. This example creates a loopback to generate a test route to track. You can use as well static routes with network statements to generate routes:

```bash
ssh -o BatchMode=yes -o StrictHostKeyChecking=no "$nva_pip_ip" <<EOF
config t
    no ip domain lookup
    interface Loopback0
        ip address 1.1.1.1 255.255.255.255
    router bgp ${nva_asn}
        network 1.1.1.1 mask 255.255.255.255
        neighbor $rs_ip1 remote-as $rs_asn
        neighbor $rs_ip1 ebgp-multihop 2
        neighbor $rs_ip1 update-source GigabitEthernet1
        neighbor $rs_ip1 timers 5 15
        neighbor $rs_ip2 remote-as $rs_asn
        neighbor $rs_ip2 ebgp-multihop 2
        neighbor $rs_ip2 update-source GigabitEthernet1
        neighbor $rs_ip2 timers 5 15
    ip route $rs_ip1 255.255.255.255 $nva_default_gw
    ip route $rs_ip2 255.255.255.255 $nva_default_gw
end
wr mem
EOF
```

If you prefer copy/paste (for example if using Azure Bastion), you can take the lines from `config t` to `wr mem` and paste them in your Cisco NVA.

## IPsec configuration

The config for IPsec is a bit more complex, so we will be using a different approach. In this repo you can find some examples of parametrized configuration files:

- [csr_config_2tunnels_tokenized.txt](./csr_config_4tunnels_tokenized.txt): when connecting a CSR with an active/active VPN Gateway
- [csr_config_4tunnels_tokenized.txt](./csr_config_4tunnels_tokenized.txt): when connecting a CSR with two active/active VPN Gateways (possibly in different regions or Virtual WAN hubs)

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
vpngw_bgp_asn=$(az network vnet-gateway show -n $vpngw_name -g $rg --query 'bgpSettings.asn' -o tsv) && echo $vpngw_bgp_asn
vpngw_gw0_pip=$(az network vnet-gateway show -n $vpngw_name -g $rg --query 'bgpSettings.bgpPeeringAddresses[0].tunnelIpAddresses[0]' -o tsv) && echo $vpngw_gw0_pip
vpngw_gw0_bgp_ip=$(az network vnet-gateway show -n $vpngw_name -g $rg --query 'bgpSettings.bgpPeeringAddresses[0].defaultBgpIpAddresses[0]' -o tsv) && echo $vpngw_gw0_bgp_ip
vpngw_gw1_pip=$(az network vnet-gateway show -n $vpngw_name -g $rg --query 'bgpSettings.bgpPeeringAddresses[1].tunnelIpAddresses[0]' -o tsv) && echo $vpngw_gw1_pip
vpngw_gw1_bgp_ip=$(az network vnet-gateway show -n $vpngw_name -g $rg --query 'bgpSettings.bgpPeeringAddresses[1].defaultBgpIpAddresses[0]' -o tsv) && echo $vpngw_gw1_bgp_ip
echo "Extracted info for vpn gateway: Gateway0 $vpngw_gw0_pip, $vpngw_gw0_bgp_ip. Gateway1 $vpngw_gw1_pip, $vpngw_gw0_bgp_ip. ASN $vpngw_bgp_asn"
```

Now you can generate the CSR config and push it to Azure:

```bash
# Variables
nva_asn=65501
# Create CSR config in a local file and copy it to the NVA
csr_config_url="https://raw.githubusercontent.com/erjosito/azure-nvas/master/csr/csr_config_2tunnels_tokenized.txt"
config_file_csr='csr.cfg'
config_file_local='/tmp/csr.cfg'
wget $csr_config_url -O $config_file_local
sed -i "s|\*\*PSK\*\*|${vpn_psk}|g" $config_file_local
sed -i "s|\*\*GW0_Private_IP\*\*|${vpngw_gw0_bgp_ip}|g" $config_file_local
sed -i "s|\*\*GW1_Private_IP\*\*|${vpngw_gw1_bgp_ip}|g" $config_file_local
sed -i "s|\*\*GW0_Public_IP\*\*|${vpngw_gw0_pip}|g" $config_file_local
sed -i "s|\*\*GW1_Public_IP\*\*|${vpngw_gw1_pip}|g" $config_file_local
sed -i "s|\*\*BGP_ID\*\*|${nva_asn}|g" $config_file_local
ssh -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip <<EOF
config t
    file prompt quiet
EOF
scp $config_file_local ${nva_pip_ip}:/${config_file_csr}
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "copy bootflash:${config_file_csr} running-config"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "wr mem"
```

## Diagnostics

There are some commands that you can issue to check different aspects of your configuration:

```bash
# CSR diagnostics
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "show ip interface brief"  # You can use this to verify the status of IPsec tunnel interfaces too
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "show ip bgp summary"      # To verify BGP neighbor status
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "show ip route"            # IP route table
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "show ip route bgp"        # BGP routes in the route table
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "show crypto ike sa"       # IKE Security Association status
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $nva_pip_ip "show crypto ipsec sa"     # IPsec Security Association status
```
