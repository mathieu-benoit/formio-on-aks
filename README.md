# form.io on Azure Kubernetes Service (AKS)

Non official setup steps to have [form.io](https://www.form.io/) deployed on [Azure Kubernetes Service (AKS)](https://azure.microsoft.com/services/kubernetes-service/).

Consideration and disclaimer: for the official deployment of form.io on Azure, you should follow the official documentation: https://help.form.io/tutorials/deployment/azure/. Associated to this, you could also find this snippet of the `docker run` commands which have been leveraged: https://gist.github.com/travist/b4e442b94339d9d973612b4df7b2124d

Here is what we would do differently:

- use Azure CLI commands to create services instead of using the Azure portal
- host the containers on AKS instead of on pure VMs
- deploy redis as container on AKS instead of using the Azure Redis Service
- leverage the Azure Monitor for containers (monitoring) and Kured (patch OS)

ToC:

- [CosmosDB and Blob Storage provisioning](#cosmosdb-and-blob-storage-provisioning)
- [Setup AKS](#setup-aks)
- [Setup ACR for the formio/formio-files-core Docker image](#setup-acr-for-the-formioformio-files-core-docker-image)
- [Tips](#tips)
- [Deploy form.io Docker containers](#deploy-formio-docker-containers)
- [Setup Ingress Controller](#setup-ingress-controller)
- [Setup HTTPS](#setup-https)
- [Leverage Day-2 features](#leverage-day-2-features)
- [TODO / FIXME / Further considerations](#todo--fixme--further-considerations)

## CosmosDB and Blob Storage provisioning

```
location=eastus
rg=yourRgName
cosmosDb=yourCosmosDbName
storage=yourStorageName

#Create the CosmosDB
az cosmosdb create \
    --kind MongoDB \
    -g $rg \
    -n $cosmosDb \
    --locations $location=0
    
#Create the Blob storage
az storage account create \
    -l $location \
    -n $storage \
    -g $rg \
    --kind BlobStorage \
    --sku Standard_LRS \
    --https-only true \
```

## Setup AKS

```
aks=youraksname

# Create the resource group
az group create \
    -l $location \
    -n $rg

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

This part is all about making available the `formio/formio-files-core` Docker image to AKS cluster via a private Azure Container Registry (ACR). This docker image is not available publicly here: https://hub.docker.com/u/formio. According to this doc: https://help.form.io/userguide/pdfserver/#deploy-the-pdf-server you need to “have a Docker Hub account, and send that to the Form.io team so that we can provide you read access to the PDF server repository”. On that regard we will pull locally this Docker image with DockerID whitelisted and then we will push that image in a private ACR.

```
acr=youracrname
az acr create \
    -n $acr \
    -g $rg \
    -l $location \
    --sku Basic \
    --admin-enabled true

#All AKS to be able to pull images from ACR
az aks update \
    -g $rg \
    -n $aks \
    --attach-acr $acr

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

# Get the Docker fomrio-files-core image tag/version
# FIXME: shouldn't be latest, that should come from (not the right value as of 2019-10-11): curl http://files.form.io/status | jq .version -r
formioFilesCoreVersion=latest

docker login $acrServer \
    -u $acrUserName \
    -p $acrPassword

docker tag formio/formio-files-core:$formioFilesCoreVersion $acrServer/formio-files-core:$formioFilesCoreVersion

docker push $acrServer/formio-files-core:$formioFilesCoreVersion
```

## Tips

```
kubectl get nodes -o wide

kubectl get pods

kubectl scale

kubectl top nodes

kubectl get pods -n $namespace

kubectl top pods -n $namespace

kubectl get all -n $namespace

kubectl get pods -o wide -n $namespace

kubectl describe pod <pod-name> -n $namespace

kubectl logs <pod-name> -n $namespace
```

## Deploy form.io Docker containers

```
#Setup of the Docker image versins/tags
# Taken from https://hub.docker.com/_/redis
redisVersion=5.0.6
# Taken from https://hub.docker.com/r/minio/minio
# FIXME: shouldn't be latest
minioVersion=latest
# Taken from https://hub.docker.com/r/formio/formio-enterprise
# Associated release notes could be found here: https://github.com/formio/enterprise-release
# To get the latest deployed version you could run: curl http://api.form.io/status | jq .version -r
formioEnterpriseVersion=6.7.24

# Create a dedicated namespace where will reside your form.io related Docker containers
namespace=yourformiok8snamespace
kubectl create ns $namespace

kubectl config set-context \
    --current \
    --namespace $namespace

kubectl run formio-redis \
    --port 6379 \
    --image redis:$redisVersion

kubectl run formio-minio \
    --env "MINIO_ACCESS_KEY=$storage" \
    --env "MINIO_SECRET_KEY=$(az storage account keys list -g $rg -n $storage --query [0].value -o tsv)" \
    --port 9000 \
    --image minio/minio:$minioVersion \
    -- gateway azure

kubectl run formio-files-core \
    --env "FORMIO_SERVER=http://formio" \
    --env "FORMIO_PROJECT=FIXME" \
    --env "FORMIO_PROJECT_TOKEN=FIXME" \
    --env "FORMIO_PDF_PROJECT=FIXME" \
    --env "FORMIO_PDF_APIKEY=FIXME" \
    --env "FORMIO_S3_SERVER=minio" \
    --env "FORMIO_S3_REGION=$location" \
    --env "FORMIO_S3_PORT=9000" \
    --env "FORMIO_S3_BUCKET=formio" \
    --env "FORMIO_S3_KEY=$storage" \
    --env "FORMIO_S3_SECRET=$(az storage account keys list -g $rg -n $storage --query [0].value -o tsv)" \
    --env "FORMIO_VIEWER=FIXME" \
    --port 4005 \
    --image $acrServer/formio-files-core:$formioFilesCoreVersion

kubectl run formio-server \
    --env "FORMIO_FILES_SERVER=http://formio-files:4005" \
    --env "PORTAL_SECRET=FIXME" \
    --env "JWT_SECRET=FIXME" \
    --env "DB_SECRET=FIXME" \
    --env "MONGO=$(az cosmosdb list-connection-strings -n $cosmosDb -g $rg --query "connectionStrings[0].connectionString" -o tsv)" \
    --env "LICENSE=FIXME" \
    --env "ADMIN_KEY=FIXME" \
    --env "PRIMARY=1" \
    --port 3000 \
    --image formio/formio-enterprise:$formioEnterpriseVersion

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

kubectl expose deployment formio-server \
    --port 80 \
    --type ClusterIP \
    --name formio
```

## Setup Ingress Controller

```
# Install Nginx as Ingress Controller
# You need to configure Helm prior to run the command below: https://docs.microsoft.com/azure/aks/kubernetes-helm
#Make sure you have the latest version of the stable Helm repo, especially since we need the nginx-ingress version > 0.22.0
helm repo update stable
# Install Nginx
helm install -n formio-ingress stable/nginx-ingress
    
# Create a dedicated Ingress making the binding between your Service and the this Nginx Ingress Controller
echo '
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: formiodev
  annotations: 
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET,HEAD,PUT,PATCH,POST,DELETE,OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-headers: "content-type, cache-control, pragma, x-remote-token, x-allow, x-expire, x-file-token, range, range-unit"
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: "/$1"
spec:
  rules:
  - host: formio<YOUR_DNS>
    http:
      paths:
      - backend:
          serviceName: formio
          servicePort: 80
        path: /(.*)$
      - backend:
          serviceName: formio-files
          servicePort: 4005
        path: /files/(.*)$
' | kubectl create -f -
```

## Setup HTTPS

```
# You need to create a Secret with the associated .key and .crt files of your own TLS certificate
kubectl create secret tls aks-ingress-tls \
    --key aks-ingress-tls.key \
    --cert aks-ingress-tls.crt

# Update the previous Ingress definition by adding the TLS section
echo '
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: formio
  annotations: 
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET,HEAD,PUT,PATCH,POST,DELETE,OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-headers: "content-type, cache-control, pragma, x-remote-token, x-allow, x-expire, x-file-token, range, range-unit"
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: "/$1"
spec:
  tls:
  - hosts:
    - formio<YOUR_DNS>
    secretName: aks-ingress-tls
  rules:
  - host: formio<YOUR_DNS>
    http:
      paths:
      - backend:
          serviceName: formio
          servicePort: 80
        path: /(.*)$
      - backend:
          serviceName: formio-files
          servicePort: 4005
        path: /files/(.*)$
' | kubectl apply -f -
```

You could now test this setup by following those instructions: https://docs.microsoft.com/azure/aks/ingress-own-tls#test-the-ingress-configuration

## Leverage Day-2 features

```
#Lock the Resource Group where the AKS cluster and other Azure service reside to prevent it from being deleted
az group lock create \
    --lock-type CanNotDelete \
    -n CanNotDelete$name \
    -g $rg

# Enable Azure Monitor for containers
# Associated doc: https://docs.microsoft.com/azure/azure-monitor/insights/container-insights-overview
az aks enable-addons \
    -a monitoring \
    -n $aks \
    -g $rg

# Install Kured to automatically apply OS Patch update
# Associated doc: https://docs.microsoft.com/azure/aks/node-updates-kured
kuredVersion=1.2.0
kubectl apply \
    -f https://github.com/weaveworks/kured/releases/download/$kuredVersion/kured-$kuredVersion-dockerhub.yaml
```

# TODO / FIXME / Further considerations

- Watch new version of:
    - The Docker images you are using from `minio`, `redis`, `formio`, `kured`, etc.
    - The Kubernetes versions by running `az aks get-versions` and then `az aks upgrade`
- Leverage K8S YAML files or Helm chart instead of `kubectl run|expose` commands
- Leverage Terraform to deploy the Infrastructure instead of using the Azure CLI
