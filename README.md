# form.io on Azure Kubernetes Service (AKS)

Non official setup steps to have [form.io](https://www.form.io/) deployed on [Azure Kubernetes Service (AKS)](https://azure.microsoft.com/services/kubernetes-service/).

Consideration and disclaimer: for the official deployment of form.io on Azure, you should follow the official documentation: https://help.form.io/tutorials/deployment/azure/. Associated to this, you could also find this snippet of the docker run commands if you are interested: https://gist.github.com/travist/b4e442b94339d9d973612b4df7b2124d

Here is what we would do differently:

- use Azure CLI commands to create services instead of using the Azure portal
- host the containers on AKS instead on pure VMs
- deploy redis as container on AKS instead of using the Azure Redis Service
- leverage the Azure Monitor for containers add-on

ToC:

- TODO

## CosmosDB and Blob Storage provisioning

```
location=eastus
rg=rgName
cosmosDb=cosmosDbName
storage=storageName

#Create the CosmosDB
az cosmosdb create \
    --kind MongoDB \
    -g $rg \
    -n $cosmosDb \
    --locations $location=0

#+ Enable manually the Preview feature "MongoDB Aggregaton Pipeline"
#https://docs.microsoft.com/en-us/azure/cosmos-db/mongodb-feature-support#aggregation-pipeline

#Get the CosmosDB connection string
az cosmosdb list-connection-strings \
    -n $cosmosDb \
    -g $rg \
    --query "connectionStrings[0].connectionString" \
    -o tsv
    
#Create the Blob storage
az storage account create \
    -l $location \
    -n $storage \
    -g $rg \
    --kind BlobStorage \
    --sku Standard_LRS \
    --https-only true \
    
#Get Blob storage access key
az storage account keys list \
    -g $rg \
    -n $storage \
    --query [0].value \
    -o tsv
    --access-tier hot 
```

# Setup AKS

location=canadacentral
name=formiodev

# Create the resource group
az group create -l $location -n $name

#This is to prevent the group from being deleted. If you need to delete, change the word create with delete
az group lock create --lock-type CanNotDelete -n CanNotDelete$name -g $name

# Create an ACR registry
az acr create -n $name -g $name -l $location --sku Basic --admin-enabled true

# Setup the AKS cluster with Azure Monitor for containers enabled
# --node-vm-size -s   for setting the of the VMs
latestK8sVersion=$(az aks get-versions -l $location --query 'orchestrators[-1].orchestratorVersion' -o tsv)
az aks create -l $location -n $name -g $name --generate-ssh-keys -k $latestK8sVersion -c 3

# Once created (the creation could take ~10 min), get the credentials to interact with your AKS cluster
az aks get-credentials -n $name -g $name


#How to create a namespace
kubectl create ns $name

kubectl run formio-redis \
  --port 6379 \
  --image redis \
  -n $name \
  

#Grant the AKS-generated service principal pull access to ACR, the AKS cluster will be able to pull images from ACR
CLIENT_ID=$(az aks show -g $name -n $name --query "servicePrincipalProfile.clientId" -o tsv)
ACR_ID=$(az acr show -n $acr -g $name --query "id" -o tsv)
az role assignment create --assignee $CLIENT_ID --role acrpull --scope $ACR_ID

# Azure Monitor for containers
az aks enable-addons -a monitoring -n $name -g $name

# Kured
kuredVersion=1.2.0
kubectl apply -f https://github.com/weaveworks/kured/releases/download/$kuredVersion/kured-$kuredVersion-dockerhub.yaml
az aks enable-addons -a monitoring -n $name -g $name



#tips
kubectl get nodes -o wide

kubectl get pods

kubectl scale

kubectl top nodes

kubectl get pods -n $name

kubectl get all -n formiodev

kubectl get pods -o wide -n formiodev

kubectl describe pod redis-785f9d6bfb-fbdpr -n $name  #what follows the redis is the pod name

kubectl logs redis-785f9d6bfb-fbdpr -n $namec


#Manual process for image
az acr show -n $name -g $name --query loginServer -o tsv

az acr credential show -n $name -g $name --query passwords[0].value -o tsv


#Docker
sudo docker login formiodev.azurecr.io -u formiodev -p xVan1ujUefSFKsS+QekMub4z55hdggMA

sudo docker tag formio/formio-files-core:latest formiodev.azurecr.io/formio-files-core:latest

sudo docker push formiodev.azurecr.io/formio-files-core:latest




kubectl run formio-minio \
  --env "MINIO_ACCESS_KEY=addatechformiodevsa" \
  --env "MINIO_SECRET_KEY===" \
  --port 9000 \
  --image minio/minio \
  -- gateway azure






kubectl run formio-files-core \
--env "FORMIO_SERVER=http://formio" \
--env "FORMIO_PROJECT=" \
--env "FORMIO_PROJECT_TOKEN=" \
--env "FORMIO_PDF_PROJECT=http://formiodev.9a84fabf5a1740ffaac0.canadacentral.aksapp.io/whatever-rbkwkreovgbtqbf" \
--env "FORMIO_PDF_APIKEY= " \
--env "FORMIO_S3_SERVER=minio" \
--env "FORMIO_S3_PORT=9000" \
--env "FORMIO_S3_BUCKET=formio" \
--env "FORMIO_S3_KEY=addatechformiodevsa" \
--env "FORMIO_S3_SECRET===" \
--env "FORMIO_VIEWER=https://cmcloudformioviewer.azurewebsites.net" \
--port 4005 \
--image formiodev.azurecr.io/formio-files-core


kubectl run formio-server \
--env "FORMIO_FILES_SERVER=http://formio-files:4005" \
--env "PORTAL_SECRET= " \
--env "JWT_SECRET= " \
--env "DB_SECRET= " \
--env "MONGO= " \
--env "LICENSE=.. " \
--env "ADMIN_KEY=" \
--env "PRIMARY=1" \
--port 3000 \
--image formio/formio-enterprise




kubectl expose deployment formio-files-core \
  --port 4005 \
  --type ClusterIP \
  --name formio-files


kubectl expose deployment formio-minio \
  --type ClusterIP \
  --name minio

kubectl expose deployment formio-redis \
  --port 6379 \
  --type ClusterIP \
  --name redis

# Maybe no Longer needed to expose it publicly since we have the Ingress Controller?
kubectl expose deployment formio-server \
  --port 80 \
  --type LoadBalancer \
  --name formio-server

az aks enable-addons \
    -g $name \
    -n $name \
    -a http_application_routing

# Enable Azure Monitor for containers
az aks enable-addons \
    -a monitoring \
    -n $name \
    -g $name

kubectl apply -f https://github.com/weaveworks/kured/releases/download/1.2.0/kured-1.2.0-dockerhub.yaml


# Further considerations

- Setup SSL on Nginx Ingress Controller based on this https://docs.microsoft.com/en-us/azure/aks/ingress-own-tls
