# form.io on Azure Kubernetes Service (AKS)

Non official setup steps to have [form.io](https://www.form.io/) deployed on [Azure Kubernetes Service (AKS)](https://azure.microsoft.com/services/kubernetes-service/).

Consideration and disclaimer: for the official deployment of form.io on Azure, you should follow the official documentation: https://help.form.io/tutorials/deployment/azure/. Associated to this, you could also find this snippet of the `docker run` commands which have been leveraged: https://gist.github.com/travist/b4e442b94339d9d973612b4df7b2124d

Here is what we would do differently:

- use Azure CLI commands to create services instead of using the Azure portal
- host the containers on AKS instead on pure VMs
- deploy redis as container on AKS instead of using the Azure Redis Service
- leverage the Azure Monitor for containers add-on

ToC:

- [CosmosDB and Blob Storage provisioning](#cosmosdb-and-blob-storage-provisioning)
- [Setup AKS](#setup-aks)
- [Setup ACR for the formio/formio-files-core Docker image](#setup-acr-for-the-formioformio-files-core-docker-image)
- [Tips](#tips)
- [Deploy form.io Docker containers](#deploy-formio-docker-containers)
- [Setup Ingress Controller](#setup-ingress-controller)
- [Enable Day-2 features](#enable-day-2-features)
- [TODO / FIXME / Further considerations](#todo--fixme--further-considerations)

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

```
location=canadacentral
rg=yourrgname
aks=youraksname

# Create the resource group
az group create \
    -l $location \
    -n $rg

#This is to prevent the group from being deleted
az group lock create \
    --lock-type CanNotDelete \
    -n CanNotDelete$name \
    -g $rg

# Setup the AKS cluster with Azure Monitor for containers enabled
latestK8sVersion=$(az aks get-versions -l $location --query 'orchestrators[-1].orchestratorVersion' -o tsv)
az aks create \
    -l $location \
    -n $aks \
    -g $rg \
    --generate-ssh-keys \
    -k $latestK8sVersion \
    -c 3 \
    -s Standard_DS2_v2

# Once created (the creation could take ~10 min), get the credentials to interact with your AKS cluster
az aks get-credentials \
    -n $aks \
    -g $rg
```

## Setup ACR for the formio/formio-files-core Docker image

```
acr=youracrname
az acr create \
    -n $acr \
    -g $rg \
    -l $location \
    --sku Basic \
    --admin-enabled true

#Grant the AKS-generated service principal pull access to ACR, the AKS cluster will be able to pull images from ACR
CLIENT_ID=$(az aks show -g $rg -n $aks --query servicePrincipalProfile.clientId -o tsv)
ACR_ID=$(az acr show -n $acr -g $rg --query id -o tsv)
az role assignment create \
    --assignee $CLIENT_ID \
    --role acrpull \
    --scope $ACR_ID

acrServer=$(az acr show \
    -n $acr \
    -g $rg \
    --query loginServer \
    -o tsv)

acrUserName=$(az acr credential show \
    -n $acr \
    -g $rg \
    --query username \
    -o tsv)
    
acrPassword=$(az acr credential show \
    -n $acr \
    -g $rg \
    --query passwords[0].value \
    -o tsv)

#you need here to ssh to the machine who previously pulled the formio/formio-files-core Docker image (DockerID whitelisted by form.io)

docker login $acrServer \
    -u $acrUserName \
    -p $acrPassword

docker tag formio/formio-files-core:latest $acrServer/formio-files-core:latest

docker push $acrServer/formio-files-core:latest
```

## Tips

```
kubectl get nodes -o wide

kubectl get pods

kubectl scale

kubectl top nodes

kubectl get pods -n $namespace

kubectl get all -n $namespace

kubectl get pods -o wide -n $namespace

kubectl describe pod <pod-name> -n $namespace

kubectl logs <pod-name> -n $namespace
```

## Deploy form.io Docker containers

```
# Create a dedicated namespace where will reside your form.io related Docker containers
namespace=yourformiok8snamespace
kubectl create ns $namespace

kubectl run formio-redis \
    --port 6379 \
    --image redis \
    -n $namespace

kubectl run formio-minio \
    --env "MINIO_ACCESS_KEY=FIXME" \
    --env "MINIO_SECRET_KEY=FIXME" \
    --port 9000 \
    --image minio/minio \
    -n $namespace \
    -- gateway azure

kubectl run formio-files-core \
    --env "FORMIO_SERVER=http://formio" \
    --env "FORMIO_PROJECT=FIXME" \
    --env "FORMIO_PROJECT_TOKEN=FIXME" \
    --env "FORMIO_PDF_PROJECT=FIXME" \
    --env "FORMIO_PDF_APIKEY=FIXME" \
    --env "FORMIO_S3_SERVER=minio" \
    --env "FORMIO_S3_PORT=9000" \
    --env "FORMIO_S3_BUCKET=formio" \
    --env "FORMIO_S3_KEY=FIXME" \
    --env "FORMIO_S3_SECRET=FIXME" \
    --env "FORMIO_VIEWER=FIXME" \
    --port 4005 \
    --image $acrServer/formio-files-core \
    -n $namespace

kubectl run formio-server \
    --env "FORMIO_FILES_SERVER=http://formio-files:4005" \
    --env "PORTAL_SECRET=FIXME" \
    --env "JWT_SECRET=FIXME" \
    --env "DB_SECRET=FIXME" \
    --env "MONGO=FIXME" \
    --env "LICENSE=FIXME" \
    --env "ADMIN_KEY=FIXME" \
    --env "PRIMARY=1" \
    --port 3000 \
    --image formio/formio-enterprise \
    -n $namespace

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

# FIXME: Maybe no Longer needed to expose it publicly since we have the Ingress Controller?
kubectl expose deployment formio-server \
    --port 80 \
    --type LoadBalancer \
    --name formio-server
```

## Setup Ingress Controller

```
# Enable HTTP Application routing to get Nginx as Ingress Controller
az aks enable-addons \
    -g $rg \
    -n $aks \
    -a http_application_routing
    
# Enable CORS with the nginx Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/mathieu-benoit/formio-on-aks/master/ingress.yaml
```

## Enable Day-2 features

```
# Enable Azure Monitor for containers
# Associated doc: https://docs.microsoft.com/en-us/azure/azure-monitor/insights/container-insights-overview
az aks enable-addons \
    -a monitoring \
    -n $aks \
    -g $rg

# Install Kured to automatically apply OS Patch update
# Associated doc: https://docs.microsoft.com/en-us/azure/aks/http-application-routing
kuredVersion=1.2.0
kubectl apply \
    -f https://github.com/weaveworks/kured/releases/download/$kuredVersion/kured-$kuredVersion-dockerhub.yaml
```

# TODO / FIXME / Further considerations

- Setup SSL on Nginx Ingress Controller based on this https://docs.microsoft.com/en-us/azure/aks/ingress-own-tls
- Don't use `latest` tag for the Docker images you are leveraging here
- Watch new version of:
    - The Docker images you are using from `minio`, `redis`, `formio`, `kured`, etc.
    - The Kubernetes versions by running `az aks get-versions` and then `az aks upgrade`
