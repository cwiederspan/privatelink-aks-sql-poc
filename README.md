# Azure Private Endpoint Test

```bash

export NAME=cdw-kubernetes-20210309
export LOCATION=westus2

az feature register --namespace Microsoft.ContainerService -n UserAssignedIdentityPreview

az group create -n $NAME -l $LOCATION

az network vnet create  \
-g $NAME \
-n $NAME \
-l $LOCATION \
--address-prefix 10.10.0.0/16

az network vnet subnet create \
--address-prefixes 10.10.1.0/24 \
-n cluster-subnet \
-g $NAME \
--vnet-name $NAME

az network vnet subnet create \
--address-prefixes 10.10.2.0/24 \
-n data-subnet \
-g $NAME \
--vnet-name $NAME \
--disable-private-endpoint-network-policies true


# Create a user-assignable identity
az identity create --name $NAME-sp --resource-group $NAME

export USER_IDENTITY=/subscriptions/b9c770d1-cde9-4da3-ae40-95ce1a4fac0c/resourcegroups/cdw-kubernetes-20210309/providers/Microsoft.ManagedIdentity/userAssignedIdentities/cdw-kubernetes-20210309-sp

# Save the Subnet IDS
export CLUSTER_SUBNET_ID=/subscriptions/b9c770d1-cde9-4da3-ae40-95ce1a4fac0c/resourceGroups/cdw-kubernetes-20210309/providers/Microsoft.Network/virtualNetworks/cdw-kubernetes-20210309/subnets/cluster-subnet
export DATA_SUBNET_ID=/subscriptions/b9c770d1-cde9-4da3-ae40-95ce1a4fac0c/resourceGroups/cdw-kubernetes-20210309/providers/Microsoft.Network/virtualNetworks/cdw-kubernetes-20210309/subnets/data-subnet


# Create the AKS Cluster
az aks create \
--resource-group $NAME \
--name $NAME \
--location $LOCATION \
--kubernetes-version 1.19.7 \
--node-count 1 \
--network-plugin kubenet \
--generate-ssh-keys \
--enable-managed-identity \
--vnet-subnet-id=$CLUSTER_SUBNET_ID \
--assign-identity $USER_IDENTITY


# Create the SQL Server
az sql server create \
-l $LOCATION \
-g $NAME \
-n $NAME-svr \
-u sqlsa \
-p CH@ng3M3! \
--enable-public-network false

export SQL_ID=/subscriptions/b9c770d1-cde9-4da3-ae40-95ce1a4fac0c/resourceGroups/cdw-kubernetes-20210309/providers/Microsoft.Sql/servers/cdw-kubernetes-20210309-svr


# Create the SQL Database
az sql db create \
-n $NAME-db \
-g $NAME \
-s $NAME-svr \
--compute-mode Serverless \
--sample-name AdventureWorksLT \
-e GeneralPurpose \
-f Gen5 \
-c 2 \
--auto-pause-delay 60


# Create the Private Endpoint
az network private-endpoint create \
-g $NAME \
-n $NAME-pe \
-l $LOCATION \
--subnet $DATA_SUBNET_ID \
--group-id sqlServer \
--private-connection-resource-id $SQL_ID \
--connection-name sql-connection \
--manual-request false


# Create the Private DNS Zone
az network private-dns zone create \
-g $NAME \
-n "privatelink.database.windows.net"

az network private-dns link vnet create \
-g $NAME \
--virtual-network $NAME \
--zone-name "privatelink.database.windows.net" \
--name $NAME-link \
--registration-enabled true

az network private-endpoint dns-zone-group create \
-g $NAME \
--endpoint-name $NAME-pe \
--name $NAME-zone \
--private-dns-zone "privatelink.database.windows.net" \
--zone-name zone-group


# Connect and Test the SQL Connection

az aks get-credentials -n $NAME -g $NAME --overwrite

kubectl run sql-cli --image=mcr.microsoft.com/mssql-tools -i --tty --rm

sqlcmd -S cdw-kubernetes-20210309-svr.database.windows.net -d cdw-kubernetes-20210309-db -U sqlsa -P CH@ng3M3!

SELECT CustomerID, CompanyName FROM SalesLT.Customer WHERE CustomerID < 100
GO

```
