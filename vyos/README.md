# Work in Progress

Unfortunately not able to run VyOS in my subscription

```
# publisher sentriumsl not valid for Microsoft internal subs
publisher=sentriumsl
offer=vyos-1-2-lts-on-azure
# sku=vyos-1-3
# sku=vyos-router-on-azure
# version=latest
version=$(az vm image list -p $publisher -f $offer -s $sku --all --query '[0].version' -o tsv)
nva_size=Standard_B2ms
vnet_name=branch1
vnet_prefix=10.4.1.0/24
subnet_name=nva
subnet_prefix=10.4.1.0/26
nva_ip_address=10.4.1.12
vm_name=vyos1
az vm create -n $vm_name -g $rg -l $location1 --image ${publisher}:${offer}:${sku}:${version} --security-type Standard \
        --generate-ssh-keys --admin-username $(whoami) --nsg "${vm_name}-nsg" --size $nva_size --disable-integrity-monitoring \
        --public-ip-address branch1-pip --public-ip-address-allocation static --public-ip-sku Standard --private-ip-address $nva_ip_address \
        --vnet-name $vnet_name --vnet-address-prefix $vnet_prefix --subnet nva --subnet-address-prefix $subnet_prefix -o none
# Error message: MarketplacePurchaseEligibilityFailed
```
