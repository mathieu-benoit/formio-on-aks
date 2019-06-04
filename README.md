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

# Further considerations

- Setup SSL on Nginx Ingress Controller based on this https://docs.microsoft.com/en-us/azure/aks/ingress-own-tls
