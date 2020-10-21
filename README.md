# ADF-PrivateLink

Demo of Azure Data Factory Private Link and how to restrict all accesses (Self Hosted Integration Runtime, Managed Integration Runtime, Data Factory Control Plane and data stores) though a customer private vnet in Azure

The Demo includes 3 private endpoint options:
* ADF uses public endpoint, Self Hosted Integration Runtime (SHIR) connects to the data strores through the private endpoint of the Data Stores
* ADF with Private Link enabled. SHIR connects to ADF through the vnet and the private endpoint
* ADF with Managed Virtual Network

The demo scenario includes the deployment of the following components:
* Azure vnet
* VM
* Azure SQL
* Storage Account
* Azure Data Factory

Accesses to and from the different components will be bounded to the vnet using Private Endpoints

## Quick recap on Private Link / Private Endpoint

[Azure Private Endpoint](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-overview) is a network interface that connects you privately and securely to a service powered by Azure Private Link. Private Endpoint uses a private IP address from your VNet, effectively bringing the service into your VNet.

[Azure Private Link](https://docs.microsoft.com/en-us/azure/private-link/private-link-overview/) enables you to access Azure PaaS Services (for example, Azure Storage and SQL Database) and Azure hosted customer-owned/partner services over a private endpoint in your virtual network.

Traffic between your virtual network and the service travels the Microsoft backbone network. Exposing your service to the public internet is no longer necessary



## Demo environment setup

### Azure vnet
Deploy a vnet / subnet on its specific IP range

### VM
Deploy a Windows Server VM on the vnet previously created, allowing RDP, HTTP and HTTPS inbound connections

Once the VM is deployed, install Azure Storage Explorer and SQL Server Management Studio:
* [ Azure Storage Explorer](https://azure.microsoft.com/en-us/features/storage-explorer/)
* [SQL Server Management Studio](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-ver15)


### Storage Account
Deploy a Blob Storage, ensuring that in the Networking section, you add a new private endpoint within the vnet that was previously created

### Azure SQL DB
1. Deploy a SQL DB
1. Create a Private Endpoint
    * Go to the Private Link Center
    * In the Private Link Center overview, go to *Build a private connection to a service*
    * In the Resource section, specify the Resource Type as *Microsoft.Sql/Servers*
    * In the Configuration section, attached the private link to the vnet/subnet

### Azure Data Factory
Deploy a Data Factory, without the use of the Managed vnet feature for now

## Scenario 1: Accesses to the Data Stores through their Private Endpoint

![ADF to datastores protected by private ndpoints](/img/ADF-to-datastores-with-private-endpoints.JPG)

### Demo 1. Test access to Storage Account through the private endpoint

#### DNS Resolution and Private IP
RDP to the VM and run the following commaned in Powershell
```Powershell
nslookup <storage-account-name>.blob.core.windows.net
```
You should receive a message similar to:
```Powershell
nslookup <storage621>.blob.core.windows.net
Server:  UnKnown
Address:  168.63.129.16

Name:    storage621.privatelink.blob.core.windows.net
Address:  172.17.0.5
Aliases:  storage621.blob.core.windows.net
```
>The name and IP resolved to the Private endpoint of the storage account


#### Accesses using Azure Storage Explorer
1. Launch Azure Storage Explorer on the VM
1. Connect to the Azure Storage using its connection string
1. Select connect. You should be able to connect to the storage account
1. Do the same from an Azure Storage Explorer that is not running on the vnet. Access should  be blocked

### Demo 2. Test access to the SQL DB through the private endpoint

#### DNS Resolution and Private IP
RDP to the VM and run the following commaned in Powershell
```Powershell
nslookup <sqldb-name>.blob.core.windows.net
```
You should receive a message similar to:
```Powershell
nslookup sqldb621.database.windows.net
Server:  UnKnown
Address:  168.63.129.16

DNS request timed out.
    timeout was 2 seconds.
Name:    sqldb621.privatelink.database.windows.net
Address:  172.17.0.6
Aliases:  sqldb621.database.windows.net
```
>The name and IP resolved to the Private endpoint of the SQL Server

#### Accesses using SQL Server Management Studio
1. Launch SQL Server Management Studio
1. In *Connect to Server*, enter the requested info
1. Connect
You should get access to the SQL server and see the DBs

### Demo 3: Access to the data strore through their private endpoint and the Self Hosted IR

> Note that at this point, the ADF itself is *not* on a vnet. ADF communicates to the SHIR and the SHIR connects to the data stores through the private endpoint of these datastores
1. Add a copy activity in the canvas
1. Create a Linked Service for the SQL DB
1. Fill the DB info and Test the connection
> When using the auto resolve integration runtime, the connction test fails.\
> When using the same DB parameter, aka public endpoint, with the Self Hosted Integration runtime it works.  The Private IP is resolved instead of the public endpoint

## Scenario 2: Setup a Private endpoint **on** Data Factory

We now create a Private Endpoint on the vnet for Data Factory. All accesses to the ADF Control Plane goes through this private endpoint

![ADF Private Link](/img/ADF-Private-Link.JPG)

Several communication channels are required between ADF and the vnet.
More details [here](https://docs.microsoft.com/en-us/azure/data-factory/data-factory-private-link#secure-communication-between-customer-networks-and-azure-data-factory) 

> **Supported scenarios:**\
> * Author and Monitor the Data Factory from the vnet
> * Command communications between the self-hosted IR and the Data Factory service goes securely through the vnet and Private Link\
>
> **Unsupported scenarios:**\
> * Interactive autohoring that uses a SHIR , such as test connection, browse folder, get schema, preview data, ...
> * Download of new verfsion of the SHIR with AutoUpdate\

> **Warning:**
When you create a Linked Service, make sure the credentials are stored in an Azure keyvault. Otherwise, the credentials won't work when you enable private link in ADF

Go to *ADF > Networking > Private Endpoint Connections* to enable Private Link on ADF
You can also disable the public access to ADF.\
However, note that this is applicable only to the Self-Hosted Integration Runtime. Azure Integration Runtime and SQL Server Integrsation Services (SSIS) IR.\


### Demo - DNS Resolution

Run a nslookup adf.azure.com on the VM and on a laptop to see the different resolutions

On a laptop:
```Powershell
nslookup adf.azure.com
Name:    waws-prod-am2-75a72192.cloudapp.net
Address:  13.73.158.206
Aliases:  adf.azure.com
	  adf.privatelink.azure.com
	  datafactoryv2.trafficmanager.net
	  datafactoryv2ase-weu.datafactoryv2ase-weu.p.azurewebsites.net
	  waws-prod-am2-75a72192.sip.p.azurewebsites.windows.net
```

On the VM in the vnet where ADF Private Link is enabled:
```Powershell
nslookup adf.azure.com
Server:  UnKnown
Address:  168.63.129.16

Name:    adf.privatelink.azure.com
Address:  172.17.0.7
Aliases:  adf.azure.com
```
> On the VM, only the private link fqdn is used with a resolution on the private endpoint of Data Factory

### Demo - SHIR Connection to the ADF Private Endpoint

In ADF > Networking, disable the public network access

In ADF > Networking > Private endpoint connections:
* If there is an approaved private endpoint for Data Factory, the SHIR is able to connect
* If there is no private endpoint for ADF or if the connection state is *Rejected*, the SHIR is not able to connect


## Scenario 3: Azure Data Factory Managed Virtual Network

This allows you to deploy an Azure Integration Runtime (aka **NOT** the SHIR) on a vnet
The Azure Integration Runtime deployed in a vnet can then leverage the private endpoints of the data stores to securely connect to them

#### Benefits of using Managed Virtual Network:
* With a Managed Virtual Network, you can offload the burden of managing the Virtual Network to Azure Data Factory. You don't need to create a subnet for Azure Integration Runtime that could eventually use many private IPs from your Virtual Network and would require prior network infrastructure planning.
* It does not require deep Azure networking knowledge to do data integrations securely. Instead getting started with secure ETL is much simplified for data engineers.
* Managed Virtual Network along with Managed private endpoints protects against data exfiltration

> Currently, the managed VNet is only supported in the same region as Azure Data Factory region.

![ADF Private Link](/img/ADF-Managed-vnet.JPG)

The creation/configuration of Managed Private Endpoint is done from the ADF/Author/Management console\
The private endpoint will be directly created on the target resource (SQl, Storage, Cosmos, etc ...) and will be accessible from the Data Factory through the use of an Azure IR integrated with the managed vnet.\
Neither the managed vnet nor these "ADF Managed Private Enpoints" are visible as Resources in the Azure Portal or in the "Private Link Console".\
The private endpoints are visible in the ADF Authoring portal and on the associated target resources.

1. Go to *Managed Private endpoints > New*
1. Create a Private Endpoint on a target 
1. Once the managed private endpoint is provisioned, it has to be apprved on the target resource
1. Create an Integration Runtime of type Azure
1. In the *"Azure Integration Runtime config"*, enable the *"Virtual network configuration"*
1. Create a Linked Service
1. In the *"New linked service"* console, chose the Azure Managed Integration Runtime that is on the Managed virtual network
1. Test the connection and Create the Linked Service
1. Create a dataset using the LKinked Service associated to the private endpoint
