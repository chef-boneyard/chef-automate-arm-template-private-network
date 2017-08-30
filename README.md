Chef Automate Azure Marketplace ARM template (internal networking)
==================================================================
This repo contains an example ARM template that has been customized from the one that you download from the portal when stepping through the UI wizard in the Azure Portal, and allows deployment of the Chef Automate Marketplace image in situations where you do not wish the image to have a public IP address assigned.

In this scenario, you would typically have a Virtual Network and associated Subnets defined, usually these are managed in their own Resource Group.  Organizations setup in this way usually have a VPN or ExpressRoute between their premises and Azure cloud.

# Overview of template changes

The template was customized by following the process below:

## Changes

- The `publicIPAddress` resource was removed from the template
- The NIC resource had a dependency on the `publicIPAddress` resource and this was removed
- The NIC resource contains a definition for `publicIPAddress` under properties/ipConfigurations - this was removed
- In the properties/ipConfigurations section of the NIC - the `privateIPAllocationMethod` was set to "Static" and an IP address was added via the `privateIPAddress` property:

```json
    "privateIPAllocationMethod": "Static",
    "privateIPAddress": "10.128.1.100",
```

_NB: Valid values for privateIPAllocationMethod are "Static" or "Dynamic".  If Dynamic is selected, a privateIPAddress must not be provided._

- Add a `parameter('fqdn')` to capture the service FQDN of the new machine (example value: `chef-automate.corp.myorg.com`)

```json
    "fqdn": {
        "type": "string",
        "metadata": {
          "description": "Internal FQDN for this machine"
        }
    }
```

- As we can no longer reference the publicIPAddress from the scripts and other resources in the template, we need to replace all instances of:

```
    reference('PublicIPAddressSetup').outputs.fqdn.value
```
with:
```
    properties('fqdn')
```

- The following 4 parameters can be removed from the parameters section of the template:
```
    publicIPAddressName
    publicIPAddressResourceGroup
    publicIPDnsName
    publicIPNewOrExisting
```

- As we no longer require a reference to the publicIPAddress deployment, we can remove the following variable:

```
"publicIPAddress": "[concat(parameters('baseUrl'), '/publicipaddress_', parameters('publicIPNewOrExisting'), '.json')]",
```

## Deployment

The template can be deployed by cloning this repository, customizing the parameters.json file and then using the following command to create a new resource group (an empty resource is required to deploy Azure Marketplace images) using the Azure CLI v2 as follows:

```bash
az login
az account set --subscription <your-subscription-guid> # optional
az group create --name sp-chefautomate-vm --location westeurope
az group deployment create -g sp-chefautomate-vm --template-file "azuredeploy.json" --parameters "@parameters.json"
```