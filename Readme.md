# Lab - Azure VPN Gateway Scenario: Active-Active gateways with highly available connections, using Palo Alto Firewalls

## Intro

The goal of this lab is to demonstrate and validate traffic flow between a simulated on-premises environment, running Palo Alto Networks firewall appliances at its edge, and an Azure cloud environment using self-managed hub and spoke networking. Dual redundancy is the target model of connectivity between the environments. See: ([Azure VPN Gateways: active-active dual redundancy](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-highlyavailable#dual-redundancy-active-active-vpn-gateways-for-both-azure-and-on-premises-networks))

### Lab Diagram

The lab uses 1 VNET to simulate the on-premises environment where all Internet-bound traffic is routed through a pair Palo Alto firewalls at the edge. The cloud environment is comprised of a simple self-managed hub and spoke environment, with a shared VPN gateway at the hub. The Palo Alto firewalls are in a simulated Active-Active HA operating model, and the Azure VPN Gateway is configured as active-active for high availability as well. The on-premises environment and the cloud environment are connected to each other using site-to-site VPN and BGP. 

Azure Bastion is used for management access to the Client VMs in the on-prem and cloud environments. To minimize costs, a single Azure Bastion Basic SKU instance is created in a separate VNet, and directly peered to all VNets with VMs to be managed.

Also to minimize costs, the VMs are configured with an auto-shutdown schedule based on time of day. The exact time of day and time zone are configurable.

Below is a diagram of what you should expect to be deployed:

![net diagram](./media/Architecture.png)

eBGP adjacencies are shown in detail below:

![BGP diagram](./media/BGP-Detail.png)

### Components

- Two hub VNets, one for on-prem and one for cloud, in two different regions (default Central US and West Central US respectively), where
    - The on-prem VNet has:
        - Two [PAN VM-Series Azure VMs](https://docs.paloaltonetworks.com/vm-series/10-2/vm-series-deployment/set-up-the-vm-series-firewall-on-azure) deployed in the BYOL model
            - These VMs are deployed using Palo Alto's default recommendation of 3 interfaces (Management, Untrust, and Trust)
            - UDRs are associated to the subnets of the Palo Alto VMs' network interfaces with BGP route propagation disabled (to prevent routing loops)
            - HTTPS remote management is enabled on the Management interface public IPs for your current public IP only; if you are running the deployment script in cloud shell, make sure to get the public IP of the system where you'll be logging into the PAN-OS web management interface from
            - licensing is beyond the scope of this lab, but the VMs will do what is needed to demonstrate the functionality without a license
        - A Route Server to simulate routes being exchanged inside the on-prem environment through BGP (note that Route Server speaks eBGP with its peers, so it's not a perfect representation)
            - This route server enables the Azure VNet on-premises environment to learn BGP routes from the Palo Alto Firewalls (in this example, the cloud Azure VPN Gateway is a BGP peer to the firewalls)
    - The cloud VNet has an Azure VPN Gateway deployed in Active-Active mode with BGP enabled
- Two spoke VNets in the same region as the cloud hub VNet, where:
    - The spoke VNets are peered directly to the cloud hub VNet
    - Transit between spoke VNets is achieved using the VPN Gateway; though this is not necessarily a valid practice in a live environment, depending on the circumstances, it is sufficient to provide transitive routing in this POC
- A separate VNet is deployed in the cloud region for remote access using Azure Bastion Basic SKU
    - All VNets with Azure VMs are directly peered to this Bastion VNet
    - This VNet provides no transitive routing, it is only for remote management access
- The local hub VNet has a Linux and Windows VM deployed for connectivity testing, accessible through Bastion or serial console
- The cloud spoke VNets each have a VM (one Linux, one Windows) deployed for connectivity testing, also accessible through Bastion or serial console
- The outcome of the lab will be full transit between all ends (all VMs can reach each other)
- BGP: The On-prem network's ASN is assigned to 65521. Azure Route Server currently uses ASN 65515, and the [ASN cannot be changed](https://docs.microsoft.com/en-us/azure/route-server/troubleshoot-route-server#why-does-my-nva-not-receive-routes-from-azure-route-server-even-though-the-bgp-peering-is-up). In order to avoid needing to force eBGP peers to learn certain routes from the same AS they use, the Azure VPN Gateway's ASN is configured as 64515.

## Using the lab
### Deploy the lab

It is strongly recommended to download the entire lab's contents to a folder where you'll be doing the deployment. All dependencies assume you are in that folder as the current working directory. Once the environment is deployed there is a manual step required to configure the Palo Alto Firewalls using generated config XML files.

You can open [Azure Cloud Shell (bash)](https://shell.azure.com) and run the following commands to build the entire lab, though you will need to download the Palo Alto Firewall configs (*-import.xml) in order to upload them using the PAN-OS HTTPS management console.

```bash
git clone https://github.com/davmhelm/pafw-azvpngw-full-mesh-bgp.git 
cd ./pafw-azvpngw-full-mesh-bgp
source ./lab-deploy.azcli
```

**Note:** the provisioning process will take around 60 minutes to complete.

Alternatively (recommended), you can run step-by-step to get familiar with the provisioning process and the components deployed:
```bash
git clone https://github.com/davmhelm/pafw-azvpngw-full-mesh-bgp.git 
cd ./pafw-azvpngw-full-mesh-bgp

## Prerequisites
## Login
# az login
## Select subscription to deploy in
# az account list -o table
# az account set -s "subcription_name"

#####################
## Begin Variables ##
#####################

# Username and password on all VMs
vm_username=azureuser
 vm_password=+Please_Ch4ng3_M3!

# VM Auto-shutdown time and time zone
# refer to autoshutdown.bicep for valid time zone IDs
autoshutdown_time=1700
autoshutdown_tz='UTC'

# Your current public IP, needed for remote web administration of Palo Alto Firewalls
mypip=$( curl -4 'https://api.ipify.org' -s )
if [ -v ACC_CLOUD ]; then
    echo -e "*** Detected running in Cloud Shell"
    echo -e "*** ACTION REQUIRED: Please go to 'https://api.ipify.org' in a local browser window and enter the result below."
    read -p "mypip: " mypip 
fi

# Define "on-prem" network environment
localSite_rg=PANBgpVpnLab-local-rg
localSite_region=centralus
localSite_vnet_name=onprem-vnet
localSite_vnet_addressSpaceCidr=10.0.0.0/19
localSite_vnet_PaMgmtSubnetCidr=10.0.0.0/24
localSite_vnet_PaUntrustSubnetCidr=10.0.1.0/24
localSite_vnet_PaTrustSubnetCidr=10.0.2.0/24
localSite_vnet_RouteServerSubnetCidr=10.0.3.0/24
localSite_vnet_VmSubnetCidr=10.0.4.0/24

# Define "on-prem" helper route server
localSite_vnet_routeServer_name=onprem-vnet-rs

# Define "on-prem" client VMs
localSite_vm1_name=LocalVM0
localSite_vm1_size=Standard_B2ms
localSite_vm1_staticIp=10.0.4.4

localSite_vm2_name=LocalVM1
localSite_vm2_size=Standard_B2ms
localSite_vm2_staticIp=10.0.4.5

# Define "on-prem" PA firewalls
localSite_fw_asn=65521
localSite_fw1_name=pafw0
localSite_fw1_size=Standard_B4ms # Must be a VM size that supports at least 3 NICs
localSite_fw1_staticIp_mgmt=10.0.0.4
localSite_fw1_staticIp_untrust=10.0.1.4
localSite_fw1_staticIp_trust=10.0.2.4
localSite_fw1_staticIp_tunnel1=10.0.2.64 # not expected to be routable outside of PAFW
localSite_fw1_staticIp_tunnel2=10.0.2.65 # not expected to be routable outside of PAFW
localSite_fw1_staticIp_loopback=10.0.2.128 # not expected to be routable outside of PAFW

localSite_fw2_name=pafw1
localSite_fw2_size=Standard_B4ms # Must be a VM size that supports at least 3 NICs
localSite_fw2_staticIp_mgmt=10.0.0.5
localSite_fw2_staticIp_untrust=10.0.1.5
localSite_fw2_staticIp_trust=10.0.2.5
localSite_fw2_staticIp_tunnel1=10.0.2.66 # not expected to be routable outside of PAFW
localSite_fw2_staticIp_tunnel2=10.0.2.67 # not expected to be routable outside of PAFW
localSite_fw2_staticIp_loopback=10.0.2.129 # not expected to be routable outside of PAFW

# Define "on-prem" firewall load balancer
localSite_lb_name=pafw-lb
localSite_lb_frontEndIp=10.0.2.224

# Define "cloud" network environment
cloudSite_rg=PANBgpVpnLab-cloud-rg
cloudSite_region=westcentralus
cloudSite_cidr_summary=10.100.0.0/20 # Used for spoke-to-spoke transitive routing. Must encompass all cloud VNet address spaces.
cloudSite_vnet_hub_name=cloud-hub-vnet
cloudSite_vnet_hub_addressSpaceCidr=10.100.0.0/22
cloudSite_vnet_hub_gatewaySubnetCidr=10.100.0.0/24

cloudSite_vnet_spoke1_name=cloud-spoke1-vnet
cloudSite_vnet_spoke1_addressSpaceCidr=10.100.4.0/22
cloudSite_vnet_spoke1_VmSubnetCidr=10.100.5.0/24

cloudSite_vnet_spoke2_name=cloud-spoke2-vnet
cloudSite_vnet_spoke2_addressSpaceCidr=10.100.8.0/22
cloudSite_vnet_spoke2_VmSubnetCidr=10.100.9.0/24

# Define "cloud" VPN gateway
cloudSite_asn=64515
cloudSite_vnet_gateway_name=cloud-hub-vnet-gw
cloudSite_vnet_gateway_psk=Msft123Msft123

# Define "cloud" client VMs
cloudSite_vm1_name=CloudVM0
cloudSite_vm1_size=Standard_B2ms
cloudSite_vm1_staticIp=10.100.5.4

cloudSite_vm2_name=CloudVM1
cloudSite_vm2_size=Standard_B2ms
cloudSite_vm2_staticIp=10.100.9.4

# Define helper Bastion
cloudSite_vnet_bastion_host=cloud-bastion
cloudSite_vnet_bastion_name=cloud-bastion-vnet
cloudSite_vnet_bastion_addressSpaceCidr=192.168.255.0/24
cloudSite_vnet_bastion_azureBastionSubnetCidr=192.168.255.0/24

# Define storage accounts for VM boot diagnostics and serial console
# recommend keeping this prefix between 1 and 12 characters 
storage_prefix=diagstorage

###################
## End Variables ##
###################

######################
## Begin Deployment ##
######################

# 1. Create Local Environment 
# 1.1 Resource Group
echo -e "*** $(date +%T%z) Creating local site - resource group $localSite_rg in $localSite_region"
az group create --name $localSite_rg --location $localSite_region --output none

# 1.2 Deploy Networking
# Create Virtual Network
echo -e "*** $(date +%T%z) Creating local site - base networking in $localSite_region"
az network vnet create --name $localSite_vnet_name --resource-group $localSite_rg --location $localSite_region --address-prefixes $localSite_vnet_addressSpaceCidr --subnet-name PA-Mgmt-Subnet --subnet-prefix $localSite_vnet_PaMgmtSubnetCidr --output none
az network vnet subnet create --name PA-Untrust-Subnet --vnet-name $localSite_vnet_name --resource-group $localSite_rg --address-prefix $localSite_vnet_PaUntrustSubnetCidr --output none
az network vnet subnet create --name PA-Trust-Subnet --vnet-name $localSite_vnet_name --resource-group $localSite_rg --address-prefix $localSite_vnet_PaTrustSubnetCidr --output none
localSite_routeServerSubnet_id=$( az network vnet subnet create --name RouteServerSubnet --vnet-name $localSite_vnet_name --resource-group $localSite_rg --address-prefix $localSite_vnet_RouteServerSubnetCidr --query "id" --output tsv )
az network vnet subnet create --name VM-Subnet --vnet-name $localSite_vnet_name --resource-group $localSite_rg --address-prefix $localSite_vnet_VmSubnetCidr --output none

# Create Route Tables
echo -e "*** $(date +%T%z) Creating local site - route tables for networking in $localSite_region"
az network route-table create --name PA-Mgmt-Subnet-rt --resource-group $localSite_rg --location $localSite_region --disable-bgp-route-propagation true --output none
az network route-table create --name PA-Untrust-Subnet-rt --resource-group $localSite_rg --location $localSite_region --disable-bgp-route-propagation true --output none
az network route-table create --name PA-Trust-Subnet-rt --resource-group $localSite_rg --location $localSite_region --disable-bgp-route-propagation true --output none

# Create Network Security Group to restrict web admin traffic from your current public IP
echo -e "*** $(date +%T%z) Creating local site - network security groups for networking in $localSite_region"
az network nsg create --name PA-Mgmt-Subnet-nsg --resource-group $localSite_rg --location $localSite_region --output none
az network nsg rule create --name AllowHttpsWebManagementIn --resource-group $localSite_rg --nsg-name PA-Mgmt-Subnet-nsg --direction Inbound --priority 100 --protocol Tcp --source-address-prefixes $mypip/32 --source-port-ranges '*' --destination-address-prefixes '*' --destination-port-ranges '443' --description "Allow HTTPS Remote Management" --output none

# Create Network Security Group to permit IKE VPN traffic between untrust interfaces and Azure VPN Gateways
# (Rules to be added in a later step)
az network nsg create --name PA-Untrust-Subnet-nsg --resource-group $localSite_rg --location $localSite_region --output none

# Create Network Security Group to permit forwarded traffic to client VMs in either region, and for traffic to/from internet
az network nsg create --name PA-Trust-Subnet-nsg --resource-group $localSite_rg --location $localSite_region --output none
az network nsg rule create --name AllowTrafficToInternet --resource-group $localSite_rg --nsg-name PA-Trust-Subnet-nsg --direction Inbound --priority 100 --protocol '*' --source-address-prefixes VirtualNetwork --source-port-ranges '*' --destination-address-prefixes Internet --destination-port-ranges '*' --description "Permit forwarding of Internet-bound traffic" --output none
az network nsg rule create --name AllowTrafficToRemoteSites --resource-group $localSite_rg --nsg-name PA-Trust-Subnet-nsg --direction Inbound --priority 200 --protocol '*' --source-address-prefixes VirtualNetwork --source-port-ranges '*' --destination-address-prefixes $cloudSite_cidr_summary --destination-port-ranges '*' --description "Permit forwarding of traffic bound for remote site" --output none
az network nsg rule create --name AllowTrafficFromInternet --resource-group $localSite_rg --nsg-name PA-Trust-Subnet-nsg --direction Outbound --priority 100 --protocol '*' --source-address-prefixes Internet --source-port-ranges '*' --destination-address-prefixes VirtualNetwork --destination-port-ranges '*' --description "Permit forwarding of reply traffic from Internet" --output none
az network nsg rule create --name AllowTrafficFromRemoteSites --resource-group $localSite_rg --nsg-name PA-Trust-Subnet-nsg --direction Outbound --priority 200 --protocol '*' --source-address-prefixes $cloudSite_cidr_summary --source-port-ranges '*' --destination-address-prefixes VirtualNetwork --destination-port-ranges '*' --description "Permit forwarding of traffic coming from remote site" --output none

# Update Subnets
az network vnet subnet update --name Pa-Mgmt-Subnet --vnet-name $localSite_vnet_name --resource-group $localSite_rg --route-table PA-Mgmt-Subnet-rt --network-security-group PA-Mgmt-Subnet-nsg --output none
az network vnet subnet update --name Pa-Untrust-Subnet --vnet-name $localSite_vnet_name --resource-group $localSite_rg --route-table PA-Untrust-Subnet-rt --network-security-group PA-Mgmt-Subnet-nsg --output none
az network vnet subnet update --name Pa-Trust-Subnet --vnet-name $localSite_vnet_name --resource-group $localSite_rg --route-table PA-Trust-Subnet-rt --network-security-group PA-Trust-Subnet-nsg --output none

# Create Load Balancer for NVA VMs
echo -e "*** $(date +%T%z) Creating local site - Internal load balancer for Palo Alto Firewalls in $localSite_region"
az network lb create --name $localSite_lb_name --resource-group $localSite_rg --sku Standard --private-ip-address $localSite_lb_frontEndIp --vnet-name $localSite_vnet_name --vnet-name $localSite_vnet_name --subnet Pa-Trust-Subnet --output none
az network lb address-pool create --name backendpool --resource-group $localSite_rg --lb-name $localSite_lb_name --output none
az network lb probe create --name myHealthProbe --resource-group $localSite_rg --lb-name $localSite_lb_name --protocol tcp --port 22 --output none
az network lb rule create --name myHaPortsRule --resource-group $localSite_rg --lb-name $localSite_lb_name --protocol All --frontend-port 0 --backend-port 0 --backend-pool-name backendpool --probe-name myHealthProbe --output none

# 1.3 Deploy Palo Alto Firewalls
echo -e "*** $(date +%T%z) Creating local site - Palo Alto Firewalls in $localSite_region"
az vm image terms accept --urn paloaltonetworks:vmseries-flex:byol:latest --output none

# Create public IPs
localSite_fw1_publicIp_mgmt=$( az network public-ip create --name $localSite_fw1_name-mgmt-pip --resource-group $localSite_rg --location $localSite_region --idle-timeout 4 --sku Standard --query "publicIp.ipAddress" --output tsv )
localSite_fw1_publicIp_untrust=$( az network public-ip create --name $localSite_fw1_name-untrust-pip --resource-group $localSite_rg --location $localSite_region --idle-timeout 4 --sku Standard --query "publicIp.ipAddress" --output tsv )
localSite_fw2_publicIp_mgmt=$( az network public-ip create --name $localSite_fw2_name-mgmt-pip --resource-group $localSite_rg --location $localSite_region --idle-timeout 4 --sku Standard --query "publicIp.ipAddress" --output tsv )
localSite_fw2_publicIp_untrust=$( az network public-ip create --name $localSite_fw2_name-untrust-pip --resource-group $localSite_rg --location $localSite_region --idle-timeout 4 --sku Standard --query "publicIp.ipAddress" --output tsv )

# Create network interfaces
az network nic create --name $localSite_fw1_name-eth0-mgmt-nic --resource-group $localSite_rg --vnet-name $localSite_vnet_name --subnet PA-Mgmt-Subnet --private-ip-address $localSite_fw1_staticIp_mgmt --public-ip-address $localSite_fw1_name-mgmt-pip --output none
az network nic create --name $localSite_fw1_name-eth1-untrust-nic --resource-group $localSite_rg --vnet-name $localSite_vnet_name --subnet PA-Untrust-Subnet --private-ip-address $localSite_fw1_staticIp_untrust --public-ip-address $localSite_fw1_name-untrust-pip --ip-forwarding true --output none
az network nic create --name $localSite_fw1_name-eth2-trust-nic --resource-group $localSite_rg --vnet-name $localSite_vnet_name --subnet PA-Trust-Subnet --private-ip-address $localSite_fw1_staticIp_trust --ip-forwarding true --lb-name $localSite_lb_name --lb-address-pools backendpool --output none
az network nic create --name $localSite_fw2_name-eth0-mgmt-nic --resource-group $localSite_rg --vnet-name $localSite_vnet_name --subnet PA-Mgmt-Subnet --private-ip-address $localSite_fw2_staticIp_mgmt --public-ip-address $localSite_fw2_name-mgmt-pip --output none
az network nic create --name $localSite_fw2_name-eth1-untrust-nic --resource-group $localSite_rg --vnet-name $localSite_vnet_name --subnet PA-Untrust-Subnet --private-ip-address $localSite_fw2_staticIp_untrust --public-ip-address $localSite_fw2_name-untrust-pip --ip-forwarding true --output none
az network nic create --name $localSite_fw2_name-eth2-trust-nic --resource-group $localSite_rg --vnet-name $localSite_vnet_name --subnet PA-Trust-Subnet --private-ip-address $localSite_fw2_staticIp_trust --ip-forwarding true --lb-name $localSite_lb_name --lb-address-pools backendpool --output none

# Create NVA VMs
az vm create --name $localSite_fw1_name --resource-group $localSite_rg --size $localSite_fw1_size --nics $localSite_fw1_name-eth0-mgmt-nic $localSite_fw1_name-eth1-untrust-nic $localSite_fw1_name-eth2-trust-nic --image paloaltonetworks:vmseries-flex:byol:latest --admin-username $vm_username --admin-password $vm_password --no-wait --output none
az vm create --name $localSite_fw2_name --resource-group $localSite_rg --size $localSite_fw2_size --nics $localSite_fw2_name-eth0-mgmt-nic $localSite_fw2_name-eth1-untrust-nic $localSite_fw2_name-eth2-trust-nic --image paloaltonetworks:vmseries-flex:byol:latest --admin-username $vm_username --admin-password $vm_password --no-wait --output none

# 1.4 Deploy client VMs
echo -e "*** $(date +%T%z) Creating local site - client VMs in $localSite_region"
# Create network interfaces
az network nic create --name $localSite_vm1_name-nic --resource-group $localSite_rg --vnet-name $localSite_vnet_name --subnet VM-Subnet --private-ip-address $localSite_vm1_staticIp --output none
az network nic create --name $localSite_vm2_name-nic --resource-group $localSite_rg --vnet-name $localSite_vnet_name --subnet VM-Subnet --private-ip-address $localSite_vm2_staticIp --output none

# Create VMs
# az vm image terms accept --urn Canonical:0001-com-ubuntu-server-jammy:22_04-lts-gen2:latest --output none # not needed, no terms to accept
az vm create --name $localSite_vm1_name --resource-group $localSite_rg --size $localSite_vm1_size --nics $localSite_vm1_name-nic --image Canonical:0001-com-ubuntu-server-jammy:22_04-lts-gen2:latest --admin-username $vm_username --admin-password $vm_password --no-wait --output none
az vm create --name $localSite_vm2_name --resource-group $localSite_rg --size $localSite_vm2_size --nics $localSite_vm2_name-nic --image MicrosoftWindowsServer:WindowsServer:2022-datacenter-azure-edition-smalldisk:latest --admin-username $vm_username --admin-password $vm_password --no-wait --output none

# Install network tools on Linux VM
az vm extension set --name CustomScript --resource-group $localSite_rg --vm-name $localSite_vm1_name --publisher Microsoft.Azure.Extensions --protected-settings "{\"fileUris\": [\"https://raw.githubusercontent.com/dmauser/azure-vm-net-tools/main/script/nettools.sh\"],\"commandToExecute\": \"./nettools.sh\"}" --no-wait --output none

# Install Network Watcher Agent on Linux VM for later connectivity testing
az vm extension set --name NetworkWatcherAgentLinux --resource-group $localSite_rg --vm-name $localSite_vm1_name --publisher Microsoft.Azure.NetworkWatcher --no-wait --output none

# Enable ICMP ping on Windows VM
az vm extension set --name CustomScriptExtension --resource-group $localSite_rg --vm-name $localSite_vm2_name --publisher Microsoft.Compute --settings '{"commandToExecute": "netsh advfirewall firewall set rule dir=in name=\"File and Printer Sharing (Echo Request - ICMPv4-In)\" new enable=yes && netsh advfirewall firewall set rule dir=in name=\"File and Printer Sharing (Echo Request - ICMPv6-In)\" new enable=yes"}' --no-wait --output none

# Install Network Watcher Agent on Windows VM for later connectivity testing
az vm extension set --name NetworkWatcherAgentWindows --resource-group $localSite_rg --vm-name $localSite_vm2_name --publisher Microsoft.Azure.NetworkWatcher --no-wait --output none

# 1.5 Deploy Route Server
echo -e "*** $(date +%T%z) Creating local site - route server in $localSite_region *** CAN TAKE UP TO 20 MINUTES ***"
# Create public IP
az network public-ip create --name $localSite_vnet_routeServer_name-pip --resource-group $localSite_rg --location $localSite_region --idle-timeout 4 --sku Standard --output none

# Deploy Route Server instance
read localSite_asn localSite_routeServer_ip1 localSite_routeServer_ip2 <<< $( az network routeserver create --name $localSite_vnet_routeServer_name --resource-group $localSite_rg --location $localSite_region --public-ip $localSite_vnet_routeServer_name-pip --hosted-subnet $localSite_routeServerSubnet_id --query "{asn:virtualRouterAsn,ips:sort(virtualRouterIps)} | {asn:asn,ip1:ips[0],ip2:ips[1]}" --output tsv )

# Configure Route Server peerings
# Don't use --no-wait, tends to lead to provisioning failures
az network routeserver peering create --name $localSite_fw1_name-peer --resource-group $localSite_rg --routeserver $localSite_vnet_routeServer_name --peer-asn $localSite_fw_asn --peer-ip $localSite_fw1_staticIp_trust --output none
az network routeserver peering create --name $localSite_fw2_name-peer --resource-group $localSite_rg --routeserver $localSite_vnet_routeServer_name --peer-asn $localSite_fw_asn --peer-ip $localSite_fw2_staticIp_trust --output none

# 2. Create Cloud Environment

# 2.1 Resource Group
echo -e "*** $(date +%T%z) Creating cloud site - resource group $cloudSite_rg in $cloudSite_region"
az group create --name $cloudSite_rg --location $cloudSite_region --output none

# 2.2 Deploy Networking
# Create Virtual Networks
echo -e "*** $(date +%T%z) Creating cloud site - base networking in $cloudSite_region"
az network vnet create --name $cloudSite_vnet_hub_name --resource-group $cloudSite_rg --location $cloudSite_region --address-prefixes $cloudSite_vnet_hub_addressSpaceCidr --subnet-name GatewaySubnet --subnet-prefix $cloudSite_vnet_hub_gatewaySubnetCidr --output none
az network vnet create --name $cloudSite_vnet_spoke1_name --resource-group $cloudSite_rg --location $cloudSite_region --address-prefixes $cloudSite_vnet_spoke1_addressSpaceCidr --subnet-name VM-Subnet --subnet-prefix $cloudSite_vnet_spoke1_VmSubnetCidr --output none
az network vnet create --name $cloudSite_vnet_spoke2_name --resource-group $cloudSite_rg --location $cloudSite_region --address-prefixes $cloudSite_vnet_spoke2_addressSpaceCidr --subnet-name VM-Subnet --subnet-prefix $cloudSite_vnet_spoke2_VmSubnetCidr --output none

# Create Route Table
echo -e "*** $(date +%T%z) Creating cloud site - route tables for networking in $cloudSite_region"
az network route-table create --name VM-Subnet-rt --resource-group $cloudSite_rg --location $cloudSite_region --output none
az network route-table route create --name Transitive-via-Gateway --route-table-name VM-Subnet-rt --resource-group $cloudSite_rg --address-prefix $cloudSite_cidr_summary --next-hop-type VirtualNetworkGateway --output none

# Update Subnets
az network vnet subnet update --name VM-Subnet --vnet-name $cloudSite_vnet_spoke1_name --resource-group $cloudSite_rg --route-table VM-Subnet-rt --output none
az network vnet subnet update --name VM-Subnet --vnet-name $cloudSite_vnet_spoke2_name --resource-group $cloudSite_rg --route-table VM-Subnet-rt --output none

# 2.3 Deploy VPN Gateway
echo -e "*** $(date +%T%z) Creating cloud site - VPN gateway in $cloudSite_region -- *** CAN TAKE UP TO 45 MINUTES ***"
# Create public IPs 
az network public-ip create --name $cloudSite_vnet_gateway_name-pip1 --resource-group $cloudSite_rg --location $cloudSite_region --idle-timeout 4 --sku Standard --output none
az network public-ip create --name $cloudSite_vnet_gateway_name-pip2 --resource-group $cloudSite_rg --location $cloudSite_region --idle-timeout 4 --sku Standard --output none

# Create Gateway instances
read cloudSite_vpngw_privateIp1 cloudSite_vpngw_publicIp1 cloudSite_vpngw_privateIp2 cloudSite_vpngw_publicIp2 <<< $( az network vnet-gateway create --name $cloudSite_vnet_gateway_name --resource-group $cloudSite_rg --location $cloudSite_region --vnet $cloudSite_vnet_hub_name --gateway-type VPN --vpn-gateway-generation Generation2 --sku VpnGw2  --public-ip-addresses $cloudSite_vnet_gateway_name-pip1 $cloudSite_vnet_gateway_name-pip2 --asn $cloudSite_asn --query "vnetGateway.bgpSettings | bgpPeeringAddresses | {private1:[0].defaultBgpIpAddresses[0],public1:[0].tunnelIpAddresses[0],private2:[1].defaultBgpIpAddresses[0],public2:[1].tunnelIpAddresses[0]}" --output tsv )
# --no-wait is not helpful here, remainder of networking steps depend on gateway being in ProvisioningState: Succeeded

# Create Peerings (must happen after gateway is created)
echo -e "*** $(date +%T%z) Creating cloud site - vnet peerings in $cloudSite_region"
az network vnet peering create --name hub-to-spoke1 --resource-group $cloudSite_rg --vnet-name $cloudSite_vnet_hub_name --remote-vnet $cloudSite_vnet_spoke1_name --allow-forwarded-traffic --allow-vnet-access --allow-gateway-transit --output none
az network vnet peering create --name spoke1-to-hub --resource-group $cloudSite_rg --vnet-name $cloudSite_vnet_spoke1_name --remote-vnet $cloudSite_vnet_hub_name --allow-forwarded-traffic --allow-vnet-access --use-remote-gateways --output none
az network vnet peering create --name hub-to-spoke2 --resource-group $cloudSite_rg --vnet-name $cloudSite_vnet_hub_name --remote-vnet $cloudSite_vnet_spoke2_name --allow-forwarded-traffic --allow-vnet-access --allow-gateway-transit --output none
az network vnet peering create --name spoke2-to-hub --resource-group $cloudSite_rg --vnet-name $cloudSite_vnet_spoke2_name --remote-vnet $cloudSite_vnet_hub_name --allow-forwarded-traffic --allow-vnet-access --use-remote-gateways --output none

# Create Local Gateways for Palo Alto firewalls
az network local-gateway create --name $localSite_fw1_name --resource-group $cloudSite_rg --location $cloudSite_region --gateway-ip-address $localSite_fw1_publicIp_untrust --asn $localSite_fw_asn --bgp-peering-address $localSite_fw1_staticIp_loopback --output none
az network local-gateway create --name $localSite_fw2_name --resource-group $cloudSite_rg --location $cloudSite_region --gateway-ip-address $localSite_fw2_publicIp_untrust --asn $localSite_fw_asn --bgp-peering-address $localSite_fw2_staticIp_loopback --output none

# Create VPN Connections
echo -e "*** $(date +%T%z) Creating cloud site - VPN connections to local site Palo Alto Firewalls in $cloudSite_region"
az network vpn-connection create --name $localSite_fw1_name-connection --resource-group $cloudSite_rg --vnet-gateway1 $cloudSite_vnet_gateway_name --local-gateway2 $localSite_fw1_name --enable-bgp --shared-key $cloudSite_vnet_gateway_psk --output none
az network vpn-connection create --name $localSite_fw2_name-connection --resource-group $cloudSite_rg --vnet-gateway1 $cloudSite_vnet_gateway_name --local-gateway2 $localSite_fw2_name --enable-bgp --shared-key $cloudSite_vnet_gateway_psk --output none

# Configure phase1/phase2 policy on VPN connections
az network vpn-connection ipsec-policy add --connection-name $localSite_fw1_name-connection --resource-group $cloudSite_rg --ike-encryption AES128 --ike-integrity SHA256 --dh-group DHGroup2 --ipsec-encryption AES128 --ipsec-integrity SHA256 --pfs-group None --sa-lifetime 102400 --sa-max-size 27000 --output none
az network vpn-connection ipsec-policy add --connection-name $localSite_fw2_name-connection --resource-group $cloudSite_rg --ike-encryption AES128 --ike-integrity SHA256 --dh-group DHGroup2 --ipsec-encryption AES128 --ipsec-integrity SHA256 --pfs-group None --sa-lifetime 102400 --sa-max-size 27000 --output none

# Add NSG rule to permit connection attempts from Azure VPN Gateways to Palo Alto Firewalls
echo -e "*** $(date +%T%z) Updating local site - network security group for Palo Alto Firewall Untrust interfaces in $localSite_region"
az network nsg rule create --name AllowTrafficFromAzureVpnGateways --resource-group $localSite_rg --nsg-name PA-Untrust-Subnet-nsg --direction Inbound --priority 100 --protocol Udp --source-address-prefixes $cloudSite_vpngw_publicIp1/32 $cloudSite_vpngw_publicIp2/32 --source-port-ranges '*' --destination-address-prefixes '*' --destination-port-ranges 500 4500 --description "Permit incoming IKE traffic" --output none

# 2.4 Deploy client VMs
echo -e "*** $(date +%T%z) Creating cloud site - client VMs in $cloudSite_region"
# Create network interfaces
az network nic create --name $cloudSite_vm1_name-nic --resource-group $cloudSite_rg --vnet-name $cloudSite_vnet_spoke1_name --subnet VM-Subnet --private-ip-address $cloudSite_vm1_staticIp --output none
az network nic create --name $cloudSite_vm2_name-nic --resource-group $cloudSite_rg --vnet-name $cloudSite_vnet_spoke2_name --subnet VM-Subnet --private-ip-address $cloudSite_vm2_staticIp --output none

# Create VMs
az vm create --name $cloudSite_vm1_name --resource-group $cloudSite_rg --size $cloudSite_vm1_size --nics $cloudSite_vm1_name-nic --image Canonical:0001-com-ubuntu-server-jammy:22_04-lts-gen2:latest --admin-username $vm_username --admin-password $vm_password --no-wait --output none
az vm create --name $cloudSite_vm2_name --resource-group $cloudSite_rg --size $cloudSite_vm2_size --nics $cloudSite_vm2_name-nic --image MicrosoftWindowsServer:WindowsServer:2022-datacenter-azure-edition-smalldisk:latest --admin-username $vm_username --admin-password $vm_password --no-wait --output none

# Install network tools on Linux VM
az vm extension set --name CustomScript --resource-group $cloudSite_rg --vm-name $cloudSite_vm1_name --publisher Microsoft.Azure.Extensions --protected-settings "{\"fileUris\": [\"https://raw.githubusercontent.com/dmauser/azure-vm-net-tools/main/script/nettools.sh\"],\"commandToExecute\": \"./nettools.sh\"}" --no-wait --output none

# Install Network Watcher Agent on Linux VM for later connectivity testing
az vm extension set --name NetworkWatcherAgentLinux --resource-group $cloudSite_rg --vm-name $cloudSite_vm1_name --publisher Microsoft.Azure.NetworkWatcher --no-wait --output none

# Enable ICMP ping on Windows VM
az vm extension set --name CustomScriptExtension --resource-group $cloudSite_rg --vm-name $cloudSite_vm2_name --publisher Microsoft.Compute --settings '{"commandToExecute": "netsh advfirewall firewall set rule dir=in name=\"File and Printer Sharing (Echo Request - ICMPv4-In)\" new enable=yes && netsh advfirewall firewall set rule dir=in name=\"File and Printer Sharing (Echo Request - ICMPv6-In)\" new enable=yes"}' --no-wait --output none

# Install Network Watcher Agent on Windows VM for later connectivity testing
az vm extension set --name NetworkWatcherAgentWindows --resource-group $cloudSite_rg --vm-name $cloudSite_vm2_name --publisher Microsoft.Azure.NetworkWatcher --no-wait --output none

# 3. Configure Helper Resources
echo -e "*** $(date +%T%z) Creating helper resources - bastion in $cloudSite_region *** CAN TAKE UP TO 10 MINUTES"
# Azure Bastion
az network vnet create --name $cloudSite_vnet_bastion_name --resource-group $cloudSite_rg --location $cloudSite_region --address-prefixes $cloudSite_vnet_bastion_addressSpaceCidr --subnet-name AzureBastionSubnet --subnet-prefix $cloudSite_vnet_bastion_azureBastionSubnetCidr --output none
az network public-ip create --name $cloudSite_vnet_bastion_host-pip --resource-group $cloudSite_rg --location $cloudSite_region --idle-timeout 4 --sku Standard --output none
az network bastion create --name $cloudSite_vnet_bastion_host --resource-group $cloudSite_rg --location $cloudSite_region --vnet-name $cloudSite_vnet_bastion_name --public-ip-address $cloudSite_vnet_bastion_host-pip --sku Basic --output none

# Peer bastion to vnets
az network vnet peering create --name bastion-to-cloudspoke1 --resource-group $cloudSite_rg --vnet-name $cloudSite_vnet_bastion_name --remote-vnet $cloudSite_vnet_spoke1_name --allow-forwarded-traffic --allow-vnet-access --output none
az network vnet peering create --name cloudspoke1-to-bastion --resource-group $cloudSite_rg --vnet-name $cloudSite_vnet_spoke1_name --remote-vnet $cloudSite_vnet_bastion_name --allow-forwarded-traffic --allow-vnet-access --output none
az network vnet peering create --name bastion-to-cloudspoke2 --resource-group $cloudSite_rg --vnet-name $cloudSite_vnet_bastion_name --remote-vnet $cloudSite_vnet_spoke2_name --allow-forwarded-traffic --allow-vnet-access --output none
az network vnet peering create --name cloudspoke2-to-bastion --resource-group $cloudSite_rg --vnet-name $cloudSite_vnet_spoke2_name --remote-vnet $cloudSite_vnet_bastion_name --allow-forwarded-traffic --allow-vnet-access --output none

local_vnet_id=$( az network vnet show --name $localSite_vnet_name --resource-group $localSite_rg --query "id" --output tsv )
bastion_vnet_id=$( az network vnet show --name $cloudSite_vnet_bastion_name --resource-group $cloudSite_rg --query "id" --output tsv )
az network vnet peering create --name bastion-to-onprem --resource-group $cloudSite_rg --vnet-name $cloudSite_vnet_bastion_name --remote-vnet $local_vnet_id --allow-forwarded-traffic --allow-vnet-access --output none
az network vnet peering create --name onprem-to-bastion --resource-group $localSite_rg --vnet-name $localSite_vnet_name --remote-vnet $bastion_vnet_id --allow-forwarded-traffic --allow-vnet-access --output none

# VM Diagnostics for serial console
# Generate random suffixes
lowerbound=10000
upperbound=9999999999
range=$(($upperbound-$lowerbound+1))
rand1=$(($(($(($RANDOM**2+$RANDOM))%$range))+$lowerbound))
rand2=$(($(($(($RANDOM**2+$RANDOM))%$range))+$lowerbound))

# Create storage accounts
echo -e "*** $(date +%T%z) Creating helper resources - boot diagnostics storage accounts in $localSite_region and $cloudSite_region"
localSite_storageUri=$( az storage account create --name $storage_prefix$rand1 --resource-group $localSite_rg --location $localSite_region --kind StorageV2 --sku Standard_LRS --query "primaryEndpoints.blob" --output tsv )
cloudSite_storageUri=$( az storage account create --name $storage_prefix$rand2 --resource-group $cloudSite_rg --location $cloudSite_region --kind StorageV2 --sku Standard_LRS --query "primaryEndpoints.blob" --output tsv )

# Enable boot diagnostics
echo -e "*** $(date +%T%z) Creating helper resources - enabling boot diagnostics for VMs in $localSite_rg and $cloudSite_rg"
localVmIds=$( az vm list --resource-group $localSite_rg --query '[*].id' --output tsv )
cloudVmIds=$( az vm list --resource-group $cloudSite_rg --query '[*].id' --output tsv )
az vm boot-diagnostics enable --storage $localSite_storageUri --ids $localVmIds --output none
az vm boot-diagnostics enable --storage $cloudSite_storageUri --ids $cloudVmIds --output none

# Enable VM auto-shutdown to save on costs
echo -e "*** $(date +%T%z) Creating helper resources - enabling auto-shutdown schedules for VMs in $localSite_rg and $cloudSite_rg"
for vm_id in ${localVmIds[@]}; do
    vmName=${vm_id##*/}
    az deployment group create --name deploy-vm-schedule-$vmName --resource-group $localSite_rg --template-file ./autoshutdown.bicep --parameters vmName=$vmName location=$localSite_region targetVmId=$vm_id shutdownTime=$autoshutdown_time timeZone="$autoshutdown_tz" --no-wait --output none
done

for vm_id in ${cloudVmIds[@]}; do
    vmName=${vm_id##*/}
    az deployment group create --name deploy-vm-schedule-$vmName --resource-group $cloudSite_rg --template-file ./autoshutdown.bicep --parameters vmName=$vmName location=$cloudSite_region targetVmId=$vm_id shutdownTime=$autoshutdown_time timeZone="$autoshutdown_tz" --no-wait --output none
done

# 4. Configure Palo Alto Firewalls

## mapping of Palo Alto Config Template Variables to script variables
# _my_hostname_ == $localSite_fw(1,2)_name
# _eth1_privateIp_ == $localSite_fw(1,2)_staticIp_untrust
# _eth1_publicIp_ == $localSite_fw(1,2)_publicIp_untrust
# _eth2_privateIp_ == $localSite_fw(1,2)_staticIp_trust
# _tunnel1_privateIp_ == $localSite_fw(1,2)_staticIp_tunnel1
# _tunnel2_privateIp_ == $localSite_fw(1,2)_staticIp_tunnel2
# _loopback_privateIp_ == $localSite_fw(1,2)_staticIp_loopback
# _pafw_asn_ == $localSite_fw_asn
# _azVpnGw_publicIp1_ == $cloudSite_vpngw_publicIp1
# _azVpnGw_privateIp1_ == $cloudSite_vpngw_privateIp1
# _azVpnGw_publicIp2_ == $cloudSite_vpngw_publicIp2
# _azVpnGw_privateIp2_ == $cloudSite_vpngw_privateIp2
# _azVpnGw_asn_ == $cloudSite_asn
# _routeServer_privateIp1_ == $localSite_routeServer_ip1
# _routeServer_privateIp2_ == $localSite_routeServer_ip2
# _routeServer_asn_ == $localSite_asn
# _azlb_frontEndIp_ == $localSite_lb_frontEndIp

# Create first Palo Alto config
sed -e "s/_my_hostname_/$localSite_fw1_name/g" \
    -e "s/_eth1_privateIp_/$localSite_fw1_staticIp_untrust/g" \
    -e "s/_eth1_publicIp_/$localSite_fw1_publicIp_untrust/g" \
    -e "s/_eth2_privateIp_/$localSite_fw1_staticIp_trust/g" \
    -e "s/_tunnel1_privateIp_/$localSite_fw1_staticIp_tunnel1/g" \
    -e "s/_tunnel2_privateIp_/$localSite_fw1_staticIp_tunnel2/g" \
    -e "s/_loopback_privateIp_/$localSite_fw1_staticIp_loopback/g" \
    -e "s/_pafw_asn_/$localSite_fw_asn/g" \
    -e "s/_azVpnGw_publicIp1_/$cloudSite_vpngw_publicIp1/g" \
    -e "s/_azVpnGw_privateIp1_/$cloudSite_vpngw_privateIp1/g" \
    -e "s/_azVpnGw_publicIp2_/$cloudSite_vpngw_publicIp2/g" \
    -e "s/_azVpnGw_privateIp2_/$cloudSite_vpngw_privateIp2/g" \
    -e "s/_azVpnGw_asn_/$cloudSite_asn/g" \
    -e "s/_routeServer_privateIp1_/$localSite_routeServer_ip1/g" \
    -e "s/_routeServer_privateIp2_/$localSite_routeServer_ip2/g" \
    -e "s/_routeServer_asn_/$localSite_asn/g" \
    -e "s/_azlb_frontEndIp_/$localSite_lb_frontEndIp/g" \
    ./pafw-config-template.xml > ./$localSite_fw1_name-import.xml

# Create second Palo Alto firewall config
sed -e "s/_my_hostname_/$localSite_fw2_name/g" \
    -e "s/_eth1_privateIp_/$localSite_fw2_staticIp_untrust/g" \
    -e "s/_eth1_publicIp_/$localSite_fw2_publicIp_untrust/g" \
    -e "s/_eth2_privateIp_/$localSite_fw2_staticIp_trust/g" \
    -e "s/_tunnel1_privateIp_/$localSite_fw2_staticIp_tunnel1/g" \
    -e "s/_tunnel2_privateIp_/$localSite_fw2_staticIp_tunnel2/g" \
    -e "s/_loopback_privateIp_/$localSite_fw2_staticIp_loopback/g" \
    -e "s/_pafw_asn_/$localSite_fw_asn/g" \
    -e "s/_azVpnGw_publicIp1_/$cloudSite_vpngw_publicIp1/g" \
    -e "s/_azVpnGw_privateIp1_/$cloudSite_vpngw_privateIp1/g" \
    -e "s/_azVpnGw_publicIp2_/$cloudSite_vpngw_publicIp2/g" \
    -e "s/_azVpnGw_privateIp2_/$cloudSite_vpngw_privateIp2/g" \
    -e "s/_azVpnGw_asn_/$cloudSite_asn/g" \
    -e "s/_routeServer_privateIp1_/$localSite_routeServer_ip1/g" \
    -e "s/_routeServer_privateIp2_/$localSite_routeServer_ip2/g" \
    -e "s/_routeServer_asn_/$localSite_asn/g" \
    -e "s/_azlb_frontEndIp_/$localSite_lb_frontEndIp/g" \
    ./pafw-config-template.xml > ./$localSite_fw2_name-import.xml

# Install Configs
echo -e "*** $(date +%T%z) ACTION REQUIRED: Install pafw XML config files on firewalls"
echo -e ""
echo -e "For each firewall:"
echo -e "1. Browse to the HTTPS management page for each firewall"
echo -e "   $localSite_fw1_name: https://$localSite_fw1_publicIp_mgmt"
echo -e "   $localSite_fw2_name: https://$localSite_fw2_publicIp_mgmt"
echo -e "2. Sign in to the firewall"
echo -e "3. Select Device tab"
echo -e "4. Select Operations tab"
echo -e "5. Select Import Named Configuration Snapshot, and upload the named XML file in this folder corresponding to the firewall:"
echo -e "   $localSite_fw1_name-import.xml"
echo -e "   $localSite_fw2_name-import.xml"
echo -e "6. Select Load Named Configuration Snapshot, and select the XML file you previously uploaded"
echo -e "7. Select Commit (top right corner of the management portal) and commit the configuration"
echo -e ""
echo -e "NOTE! The configs are created assuming the following defaults:"
echo -e "   vm_username: azureuser"
echo -e "   vm_password: +Please_Ch4ng3_M3!"
echo -e "   cloudSite_vnet_gateway_psk: Msft123Msft123"
echo -e ""
echo -e "*** $(date +%T%z) Automated deployment complete."

####################
## End Deployment ##
####################

# See lab-examine.azcli for tests to validate environment
```

### Applying firewall configs
Below is a video that demonstrates the steps of:
1. signing into the HTTPS managment page of a Palo Alto firewall,
2. Importing, loading, and committing the generated config.xml file, and
3. Checking the status of VPN tunnels and BGP routes after the configuration is applied

https://user-images.githubusercontent.com/21045679/183094641-4545549a-3245-4a76-ad88-fd79929bd53d.mp4

## Test Connectivity

The [lab-examine](./lab-examine.azcli) script contains commands to check the effective routes on the deployed VMs, along with BGP routes advertised by/learned by the Route Server and VPN Gateway. There are also commands to test ICMP connectivity between VMs using Azure Network Watcher. 

Additionally, you can connect to the lab VMs using Azure Bastion to run your network tools of choice to validate connectivity (hping, traceroute, etc).

```bash
# Parameters
localSite_rg=PANBgpVpnLab-local-rg
cloudSite_rg=PANBgpVpnLab-cloud-rg

######################################
## Tests to examine the environment ##
######################################

# 1) Test connectivity between VMs

# get lists of NICs
localNics=$( az network nic list --resource-group $localSite_rg --query '[].name' --output tsv)
cloudNics=$( az network nic list --resource-group $cloudSite_rg --query '[].name' --output tsv)

# Get list of VM private IPs
for nicname in ${localNics[@]}; do
    privateIp=$( az network nic show --name $nicname --resource-group $localSite_rg --query 'ipConfigurations[].privateIpAddress' --output tsv )
    echo -e "$localSite_rg/$nicname private IP: $privateIp"
done
echo -e ""

for nicname in ${cloudNics[@]}; do
    privateIp=$( az network nic show --name $nicname --resource-group $cloudSite_rg --query 'ipConfigurations[].privateIpAddress' --output tsv )
    echo -e "$cloudSite_rg/$nicname private IP: $privateIp"
done

echo -e ""

# Dump VM route tables
for nicname in ${localNics[@]}; do
    echo -e "$localSite_rg/$nicname effective routes:"
    az network nic show-effective-route-table --name $nicname --resource-group $localSite_rg --output table
    echo -e ""
done

for nicname in ${cloudNics[@]}; do
    echo -e "$cloudSite_rg/$nicname effective routes:"
    az network nic show-effective-route-table --name $nicname --resource-group $cloudSite_rg --output table
    echo -e ""
done

# 2) Check BGP

# Dump Route Server Advertised routes
# https://docs.microsoft.com/en-us/cli/azure/network/routeserver?view=azure-cli-latest
rsname=$( az network routeserver list --resource-group $localSite_rg --query '[0].name' --output tsv)
rspeerings=$( az network routeserver peering list --routeserver $rsname --resource-group $localSite_rg --query '[].name' --output tsv)

echo -e "*** $localSite_rg/$rsname Route Server peerings: ***"
az network routeserver peering list --routeserver $rsname --resource-group $localSite_rg --query '[].{name:name,peerIp:peerIp,asn:Asn}' --output table
echo -e ""

for peer in ${rspeerings[@]}; do
    echo -e "*** $localSite_rg/$rsname/$peer advertised routes: **"
    az network routeserver peering list-advertised-routes --name $peer --resource-group $localSite_rg --routeserver $rsname --query "[RouteServiceRole_IN_0,RouteServiceRole_IN_1][]" --output table
    echo -e ""

    echo -e "*** $localSite_rg/$rsname/$peer learned routes: **"
    az network routeserver peering list-learned-routes --name $peer --resource-group $localSite_rg --routeserver $rsname --query "[RouteServiceRole_IN_0,RouteServiceRole_IN_1][]" --output table
    echo -e ""
done

# Dump VPN Gateway learned routes
vpngw=$( az network vnet-gateway list --resource-group $cloudSite_rg --query '[0].name' --output tsv )

echo -e "*** $cloudSite_rg/$vpngw BGP peer status ***"
az network vnet-gateway list-bgp-peer-status --name $vpngw --resource-group $cloudSite_rg --output table
echo -e ""
echo -e "*** $cloudSite_rg/$vpngw BGP learned routes ***"
az network vnet-gateway list-learned-routes --name $vpngw --resource-group $cloudSite_rg --output table
echo -e ""

# VPN Gateways advertised routes per BGP peer.
ips=$( az network vnet-gateway list-bgp-peer-status --name $vpngw --resource-group $cloudSite_rg --query 'value[].{ip:neighbor}' --output tsv )
for ip in ${ips[@]}; do
    echo -e "*** $cloudSite_rg/$vpngw advertised routes to peer $ip ***"
    az network vnet-gateway list-advertised-routes --name $vpngw --resource-group $cloudSite_rg --peer $ip --output table
    echo -e ""
done

# Azure Network Watcher 
# Shared for completeness, but some tests do not appear to work through CLI or portal
# https://docs.microsoft.com/en-us/azure/network-watcher/network-watcher-connectivity-cli
# https://docs.microsoft.com/en-us/azure/network-watcher/network-watcher-connectivity-portal

localVmIds=$( az vm list --resource-group $localSite_rg --query "[?plan.publisher!='paloaltonetworks'].id" --output tsv )
cloudVmIds=$( az vm list --resource-group $cloudSite_rg --query "[?plan.publisher!='paloaltonetworks'].id" --output tsv )

# Local VM->Cloud VM will return result Unknown
# Skip

# Cloud VM->Local VM should work
for vmid_source in ${cloudVmIds[@]}; do 
    for vmid_dest in ${localVmIds[@]}; do
        if [ "$vmid_source" != "$vmid_dest" ]; then
            echo -e "*** Testing connectivity from ${vmid_source##*/} to ${vmid_dest##*/} ***"
            az network watcher test-connectivity --source-resource $vmid_source --dest-resource $vmid_dest --protocol icmp --output table
            echo -e ""
        fi
    done
done

# Local VM->Local VM should work
for vmid_source in ${localVmIds[@]}; do 
    for vmid_dest in ${localVmIds[@]}; do
        if [ "$vmid_source" != "$vmid_dest" ]; then
            echo -e "*** Testing connectivity from ${vmid_source##*/} to ${vmid_dest##*/} ***"
            az network watcher test-connectivity --source-resource $vmid_source --dest-resource $vmid_dest --protocol icmp --output table
            echo -e ""
        fi
    done
done

# Cloud VM->Cloud VM should work
for vmid_source in ${cloudVmIds[@]}; do 
    for vmid_dest in ${cloudVmIds[@]}; do
        if [ "$vmid_source" != "$vmid_dest" ]; then
            echo -e "*** Testing connectivity from ${vmid_source##*/} to ${vmid_dest##*/} ***"
            az network watcher test-connectivity --source-resource $vmid_source --dest-resource $vmid_dest --protocol icmp --output table
            echo -e ""
        fi
    done
done

# Test connectivity of local VMs to a web address
for vmid_source in ${localVmIds[@]}; do 
    echo -e "*** Testing connectivity from ${vmid_source##*/} to web address https://www.bing.com ***"
    az network watcher test-connectivity --source-resource $vmid_source --dest-address 'https://www.bing.com' --protocol tcp --dest-port 443 --output table
    echo -e ""
done

# Test connectivity of cloud VMs to a web address
for vmid_source in ${cloudVmIds[@]}; do 
    echo -e "*** Testing connectivity from ${vmid_source##*/} to web address https://www.bing.com ***"
    az network watcher test-connectivity --source-resource $vmid_source --dest-address 'https://www.bing.com' --protocol tcp --dest-port 443 --output table
    echo -e ""
done
```

## Clean Up

Simply delete the resource groups created by lab-deploy.azcli script. You will be prompted for confirmation of each group's deletion.

```bash
az group delete --name $localSite_rg --force-deletion-types Microsoft.Compute/virtualMachines --no-wait
az group delete --name $cloudSite_rg --force-deletion-types Microsoft.Compute/virtualMachines --no-wait
```

## Thanks
Thanks are owed to the many labs I've studied to help put this together. Not an exhaustive list, but these include:
* [dmauser/azure-vm-net-tools](https://github.com/dmauser/azure-vm-net-tools)
* [jwrightazure/lab/pan-vpn-to-azurevpngw-ikev2-bgp](https://github.com/jwrightazure/lab/tree/main/pan-vpn-to-azurevpngw-ikev2-bgp)
* [jwrightazure/lab/PAN-BGP-ARS](https://github.com/jwrightazure/lab/tree/main/PAN-BGP-ARS)
* [jwrightazure/lab/VWAN-InternetAccess-Spoke-PaloAlto](https://github.com/jwrightazure/lab/tree/main/VWAN-InternetAccess-Spoke-PaloAlto)

## FAQ

### Why is the Azure Bastion VNet separate from all the other VNets? 
Couple of reasons:
1. It's less expensive to use one Bastion Basic SKU instance across the entire lab than one per-environment.
2. Having Bastion in its own VNet protects it from the 0.0.0.0/0 route advertisements being done in the on-prem VNet. 
    * An alternate approach would be to create an Azure Bastion Standard SKU instance in the cloud hub VNet and enable [IP-based connections](https://docs.microsoft.com/en-us/azure/bastion/connect-ip-address), which would allow you to connect to the VMs in the on-premises environment using their Private IPs. 
        * The lab configuration is designed to prevent this, but in a real environment you must ensure that the default route (0.0.0.0/0) is not learned from on-prem, or your ability to use the Bastion instance will be broken until you fix the routing.
        * Configuring BGP route filtering is beyond the scope of this lab, but in the context of Palo Alto I found these articles helpful:
            * [PAN Docs: How to configure BGP](https://docs.paloaltonetworks.com/pan-os/10-1/pan-os-networking-admin/bgp/configure-bgp)
            * [PAN KB: How to configure BGP route filtering](https://knowledgebase.paloaltonetworks.com/KCSArticleDetail?id=kA10g000000ClDuCAK)
            * [PAN KB: How to import/export a default route using BGP](https://knowledgebase.paloaltonetworks.com/kcSArticleDetail?id=kA10g000000CltU)
    * As of this writeup there is no way to protect the AzureBastionSubnet from learning routes that would impede its functionality/put it into an unsupported state.

### Why do we need to change the ASN away from the default on the Azure VPN Gateway (65515)?
- In some circumstances it's normal for eBGP peers to receive route advertisements with an [AS Path](https://datatracker.ietf.org/doc/html/rfc4271#section-5.1.2) containing their own ASN, and during [Phase 2 of the Decision Process](https://datatracker.ietf.org/doc/html/rfc4271#section-9.1.2) make their own decision on whether to incorporate the routes in their [Routing Information Base (RIB)](https://en.wikipedia.org/wiki/Routing_table). 
- Though it is beyond the scope of this document: with Palo Alto Firewalls, there is an optional feature enabled that causes the sender to detect the loop pre-emptively and not send over the route at all. More info:
    - [PAN Docs: Sender Side Loop Detection](https://docs.paloaltonetworks.com/pan-os/10-1/pan-os-networking-admin/bgp/configure-a-bgp-peer-with-mp-bgp-for-ipv4-or-ipv6-unicast#:~:text=when%20you%20enable%20sender%20side%20loop%20detection%2C%20the%20firewall%20will%20check%20the%20as_path%20attribute%20of%20a%20route%20in%20its%20fib%20before%20it%20sends%20the%20route%20in%20an%20update%2C%20to%20ensure%20that%20the%20peer%20as%20number%20is%20not%20on%20the%20as_path%20list.%20if%20it%20is%2C%20the%20firewall%20removes%20it%20to%20prevent%20a%20loop)
    - [PAN KB: BGP advertisements through an eBGP peer not occurring between two peers in the same AS](https://knowledgebase.paloaltonetworks.com/KCSArticleDetail?id=kA10g000000ClyqCAC)
    - [PAN Community: BGP advertising prfix to same AS it was learned from](https://live.paloaltonetworks.com/t5/general-topics/bgp-advertising-prefix-to-same-as-it-was-learned-from/m-p/145383)
