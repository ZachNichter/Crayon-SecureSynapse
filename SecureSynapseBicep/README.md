# SecureSynapseBicep Structure

This project consists of a driver Bicep template that is linked to a number of scoped Bicep templates as listed:
```
* 01-Vnet: creates VNET necessary for deployment

* 02-Vm: creates a jumpbox VM (optional)

* 03-SynapseWs: creates a Synapse workspace with managed Vnet. No provisioned reources (SQL) are created. The workspace can be locked down on the network by setting "allowAllConnections: false"

* 04-SynapseHub: creates a Synapse Private Link Hub to secure access to Synapse Studio

* 05-SynapsePE: creates private endpoints for Synapse Studio (web), dev, sql, and sqlOnDemand access points
```

The resulting deployment looks like this:

![Deployed Architecture](images/deployedArchitecture.png?raw=true "Architecture")

# Prerequisites
You **do NOT need to have an existing resource group**. The Bicep deployment will create a default resource group for you (rg-secsyn-bicep). Note that Synapse creates a managed resource group as part of the deployment process. The managed resource group is not always cleaned up if you delete your own resource group.

## Secure Credentials
In order to avoid storing passwords in the templates, you must pass a password as a parameter as shown below:
```powershell

az deployment sub create -f ./main.bicep -l "your-region"  -n "your-deployment-name"" -p CreateJumpVm=true CreateSynapseWs=true vmAdminPassword="your-vm-password" synapseAdminPassword="your-Synapse-password"
```

Template parameters specified on the command line will overwrite the defaults in the template.

# Example Calls

## Azure CLI
Clone or download the Github repo from https://github.com/vsuopys/SecureSynapse.

On your local machine, switch directory to SecureSynapseBicep.

Run the following commands:

```powershell

az login

az deployment sub create -f ./main.bicep -l "your-region"  -n "your-deployment-name"" -p CreateJumpVm=true CreateSynapseWs=true vmAdminPassword="your-vm-password" synapseAdminPassword="your-Synapse-password"
```

# Post Deployment Requirements
```
1. The jumpbox VM is configured to use AAD authentication. You will need to add yourself to the "Virtual Machine User Login" or "Virtual Machine Administrator Login" roles for this VM or you will not be able to log in with you AAD credentials.
2. Make sure the Windows OS is fully updated. You may have to check for updates multiple times after rebooting if needed.
3. You will need to configure the jumpbox VM with Microsoft endpoint protection. On the VM, go to "Settings/Account/Access Work or School". Disconnect from Microsoft Azure AD, reboot, and re-login as the local administrator account. Go back to "Settings/Account/Access Work or School" and add a new connection to Microsoft Azure AD with your Microsoft account. This will register the VM with Microsoft endpoint protection.
4. (Optional) It would be best practice to enable just in time access through the Azure Portal for this VM. You may be prompted for VPN and disk encryption which are optional.

5. Manually create a private endpoint for the default Synapse data lake account in the "privateEndpointSubnet". This will allow Synapse Studio to see files in the storage account. I will fix this in the near future.
6. Manually create a *managed* private endpoint for the default Synapse data lake account. This will allow Synapse background processes to see files in the storage account. I will fix this in the near future.

```

# Notes and Bugs
The main Bicep template (main.bicep) references several linked templates (aka. modules) that are stored locally.

The Azure storage account does not get assigned MSI privileges for the Synapse workspace. For now you have to assign this manually to your Synapse workspace in the pulldown. See the below screenshot of the storage account network configuration:

![Storage ACL Fix](images/storageACLSBug.png?raw=true "Storage ACLS")

# Parameter Specifications
These values can be overridden on the command line.

```bicep
param vnetName string = 'vnet-customer-bicep'
param vnetAddressCidr string = '172.19.0.0/16'
param defaultSubnetCidr string = '172.19.0.0/26'
param gatewaySubnetCidr string = '172.19.1.0/26'
param privateEndpointSubnetCidr string = '172.19.2.0/26'
param dataSubnetCidr string = '172.19.3.0/26'

param createJumpVm bool = true
param vmName string = 'vm'
param vmAdminName string = 'cloudsa'
param vmAdminPassword string

param createSynapseWs bool = true
param synapseAdminName string = 'adminuser'
param synapseAdminPassword string
param synapseWsName string = 'secsynbic-ws'
param allowAllConnections bool = false
param createNewStorageAccount bool = true
param storageAccountName string = 'secbicdlac'
param storageFilesystemName string = 'synapsefilesystem'
```