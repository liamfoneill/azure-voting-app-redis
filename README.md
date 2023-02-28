# Liam's Azure Voting App

This sample creates a multi-container application in an Azure Kubernetes Service (AKS) cluster. The application interface has been built using Python / Flask. The data component is using Redis.

To walk through a quick deployment of this application, see the AKS [quick start](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough?WT.mc_id=none-github-nepeters).

To walk through a complete experience where this code is packaged into container images, uploaded to Azure Container Registry, and then run in and AKS cluster, see the [AKS tutorials](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-app?WT.mc_id=none-github-nepeters).

## Demo Steps

### Set Variables for Demo
``` bash
RESOURCEGROUP="buildazure-containerdemo"
AKSNODERESOURCEGROUP='buildazure-containerdemo-aksinfra'
LOCATION="uksouth"
ACRNAME='buildingonazuredemo'
ACRSKU='Premium'
AKSCLUSTERNAME='buildingonazuredemo'
LOGANALYTICSWORKSPACEID='/subscriptions/50e40ccc-abd3-46ef-812c-a4194b9a530e/resourceGroups/buildazure-mgmt/providers/Microsoft.OperationalInsights/workspaces/buildazure-law'
AKSADMINSOBJECTID='944d0d81-a716-47a7-affa-fea5f2d87e52'
AADTENANTID='f43111b6-c9d9-4062-9aec-8bf8c345e9ab'
```
### Prepare Application for AKS
``` bash
git clone https://github.com/Azure-Samples/azure-voting-app-redis.git
cd azure-voting-app-redis
docker-compose up -d
docker images
```
### Create Container Registry
``` bash
az account set --subscription 'Online'
az group create --name $RESOURCEGROUP --location $LOCATION
az acr create --resource-group $RESOURCEGROUP --name $ACRNAME --sku $ACRSKU
az acr replication create -r $ACRNAME -l ukwest
az acr login --name $ACRNAME
az acr list --resource-group $RESOURCEGROUP --query "[].{acrLoginServer:loginServer}" --output table
docker tag mcr.microsoft.com/azuredocs/azure-vote-front:v1 $ACRNAME.azurecr.io/azure-vote-front:v1
docker push $ACRNAME.azurecr.io/azure-vote-front:v1
az acr repository list --name $ACRNAME --output table
az acr repository show-tags --name $ACRNAME --repository azure-vote-front --output table
```
### Create Kubernetes Cluster
#### Register resource providers
``` bash
az feature register --namespace "Microsoft.Compute" --name "EncryptionAtHost"
az feature show --namespace "Microsoft.ContainerService" --name "AKS-KedaPreview"

az feature register --namespace "Microsoft.ContainerService" --name "EnableWorkloadIdentityPreview"
az feature show --namespace "Microsoft.ContainerService" --name "EnableWorkloadIdentityPreview"

az feature register --namespace "Microsoft.ContainerService" --name "AKS-KedaPreview"
az feature show --namespace "Microsoft.Compute" --name "EncryptionAtHost"       
```
#### Create the Kubernetes cluster
``` bash
az aks create \
    --resource-group $RESOURCEGROUP \
    --name $AKSCLUSTERNAME \
    --node-count 3 \
    --generate-ssh-keys \
    --attach-acr $ACRNAME \
    --enable-managed-identity \
    --enable-aad --aad-admin-group-object-ids $AKSADMINSOBJECTID --aad-tenant-id $AADTENANTID \
    --enable-encryption-at-host \
    --enable-azure-rbac \
    --enable-defender \
    --attach-acr $ACRNAME \
    --enable-keda \
    --enable-addons monitoring --workspace-resource-id $LOGANALYTICSWORKSPACEID \
    --zones 1 2 3 \
    --uptime-sla \
    --node-resource-group $AKSNODERESOURCEGROUP \
    --auto-upgrade-channel stable \
    --enable-cluster-autoscaler --max-count 5 --min-count 1 \
    --enable-addons azure-keyvault-secrets-provider,azure-policy,open-service-mesh,monitoring

NB: I would like to add an app gateway to this but I need to deploy into an existing VNET for that.
```
``` bash
az aks update -n $AKSCLUSTERNAME -g $RESOURCEGROUP --attach-acr $ACRNAME

az aks install-cli

az aks get-credentials --resource-group $RESOURCEGROUP --name $AKSCLUSTERNAME

kubectl get nodes

```
### Run Application
``` bash
az acr list --resource-group $RESOURCEGROUP --query "[].{acrLoginServer:loginServer}" --output table

code azure-vote-all-in-one-redis.yaml

kubectl apply -f azure-vote-all-in-one-redis.yaml

kubectl get pods --watch

kubectl get service
```
### Scale Application
``` bash
kubectl apply -f azure-vote-hpa.yaml

kubectl get hpa
```
### Update Application
``` bash
code azure-vote/azure-vote/config_file.cfg

docker-compose up --build -d

docker tag $ACRNAME.azurecr.io/azure-vote-front:v1 $ACRNAME.azurecr.io/azure-vote-front:v2

docker push $ACRNAME.azurecr.io/azure-vote-front:v2

kubectl set image deployment azure-vote-front azure-vote-front=$ACRNAME.azurecr.io/azure-vote-front:v2
```
### Upgrade Cluster
``` bash
az aks get-upgrades --resource-group $RESOURCEGROUP --name $AKSCLUSTERNAME --output table

az aks upgrade \
    --resource-group $RESOURCEGROUP \
    --name $AKSCLUSTERNAME \
    --kubernetes-version 1.25.5

kubectl get events

az aks show --resource-group $RESOURCEGROUP --name $AKSCLUSTERNAME --output table
```
### Delete Cluster
``` bash
az group delete --name myResourceGroup --yes --no-wait
```

### Optional: Setup Grafana and Prometheus 
``` bash
az resource create --resource-group $RESOURCEGROUP --namespace microsoft.monitor --resource-type accounts --name buildingonazure --location $LOCATION --properties {}
az grafana create --name buildingonazure --resource-group $RESOURCEGROUP
```
