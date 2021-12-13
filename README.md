# Azure DNS Resolver Sample

## TL,DR
This repository can be used to deploy a sample of a highly available DNS resolver that is capable of resolving public IP addresses of private endpoint enabled resources that are not connected to or configured in your corporate network. This is to overcome the behavior described in [https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-dns](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-dns):
> Private networks already using the private DNS zone for a given type, can only connect to public resources if they don't have any private endpoint connections, otherwise a corresponding DNS configuration is required on the private DNS zone in order to complete the DNS resolution sequence. 

<br>

## Deployment
There are multiple Options to deploy this Sample.

### Option 1 - Use "Deploy to Azure" Button

You can use this "Deploy to Azure" butten to create the Sample environment in your subscription:

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fgithub.com%2Fcthoenes%2Fazure-coredns-forwarder-poc%2Freleases%2Flatest%2Fdownload%2Fmain.json" target="_blank">
    <img src="https://aka.ms/deploytoazurebutton"/>
</a>

<br>

### Option 2 - Use Azure CLI
You can deploy the Sample using a Azure CLI deployment to your subscription

To Do so please clone the repository and create a parameter file for your deployment looking similar to this example:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
      "rgName": {
        "value": "<resourceGroupName>"
      },
      "location": {
        "value": "<yourPreferredLocation>"
      },
      "adminUser": {
        "value": "azureuser"
      },
      "publicKey": {
        "value": "<sshPublicKey>"
      },
      "deployBastion": {
        "value": true
      },
      "deployResolver": {
        "value": true
      },
      "deployPrivateZone": {
        "value": true
      },
      "deployStorageAccount": {
        "value": true
      }
    }
  }
````
Afterward you can use AZ CLI to deploy to your Subscription. Make sure you are logged on and the correct subscription is set.

```shell
az deployment sub create --location <yourPreferredLocation> --template-file ./iac/main.bicep --parameters @parameterFile.json
```

## Tests after Deployment

1. Connect to your resolver vm using Azure Bation

2. From the Console start nslookup by typing:
```shell
nslookup
```

3. In nslookup type the name of the created Storage Account. Make sure you try to access the blob endpoint as this is the only private endpoint enabled subresources in this sample.
```shell
> storageaccount.blob.core.windows.net
Server:127.0.0.53
Address:127.0.0.53#53

** server can't find storageaccount.blob.core.windows.net: NXDOMAIN
```
- Ubuntu will refer to 127.0.0.1 as DNS Server and return no result for the query as it uses the Azure provided DNS in the background. The query will result in an NXDOMAIN.

4. To verify that the Azure DNS is not capable of resolving the name due to a missing entry in the Private DNS Zone type:
```shell
> server 168.63.129.16
Default server: 168.63.129.16
Address: 168.63.129.16#53
> storageaccount.blob.core.windows.net
Server:168.63.129.16
Address:168.63.129.16#53

Non-authoritative answer:
storageaccount.blob.core.windows.net canonical name = storageaccount.privatelink.blob.core.windows.net.
** server can't find storageaccount.privatelink.blob.core.windows.net: NXDOMAIN
```
- Now DNS resolution directly points to the Azure DNS Service. Requesting the blob endpoint will result in an NXDOMAIN.

5. Now point the DNS server to a public DNS Server to verify that the public DNS is capable of resolving to the storage accounts Public IP address:
```shell
> server 1.1.1.1 
Default server: 1.1.1.1
Address: 1.1.1.1#53
> storageaccount.blob.core.windows.net
Server:1.1.1.1
Address:1.1.1.1#53

Non-authoritative answer:
storageaccount.blob.core.windows.net canonical name = storageaccount.privatelink.blob.core.windows.net.
storageaccount.privatelink.blob.core.windows.net canonical name = blob.<publicname>.store.core.windows.net.
Name:blob.<publicname>.store.core.windows.net
Address: <PublicIP>
````
- The public DNS Server points to the Public IP address as expected.

6. As a last Test use the newly created Virtual Machine Scale Set running coredns. Therefore set the server to 10.0.0.200 (Loadbalancer IP) and resolve the endpoint again:
```shell
> server 10.0.0.200
Default server: 10.0.0.200
Address: 10.0.0.200#53
> storageaccount.blob.core.windows.net
Server:10.0.0.200
Address:10.0.0.200#53

Non-authoritative answer:
storageaccount.blob.core.windows.net canonical name = storageaccount.privatelink.blob.core.windows.net.
storageaccount.privatelink.blob.core.windows.net canonical name = blob.<publicname>.store.core.windows.net.
> blob.<publicname>.store.core.windows.net.
Server:10.0.0.200
Address:10.0.0.200#53

Non-authoritative answer:
Name:blob.<publicname>.store.core.windows.net
Address: <PublicIP>
```

- As configured the coredns server will use a alternate public DNS Server if the result of the forward is a NXDOMAIN. If the firewall setting on the storage account allow access it is possible to use the Public IP even though it is not configured in the Private DNS Zone.