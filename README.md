# formio-on-aks

# Introduction

# CosmosDB and Blob Storage provisioning

```
location=eastus
name=mabenoitfio

#Create the CosmosDB
az cosmosdb create \
    --kind MongoDB \
    -g $name \
    -n $name \
    --locations $location=0

#+ Enable manually the Preview feature "MongoDB Aggregaton Pipeline"
#https://docs.microsoft.com/en-us/azure/cosmos-db/mongodb-feature-support#aggregation-pipeline

#Get the CosmosDB connection string
az cosmosdb list-connection-strings \
    -n $name \
    -g $name \
    --query "connectionStrings[0].connectionString" \
    -o tsv
    
#Create the Blob storage
az storage account create \
    -l $location \
    -n $name \
    -g $name \
    --kind BlobStorage \
    --sku Standard_LRS \
    --https-only true \
    
#Get Blob storage access key
az storage account keys list \
    -g $name \
    -n $name \
    --query [0].value \
    -o tsv
    --access-tier hot 
```

# Further considerations

- Setup SSL on Nginx Ingress Controller based on this https://docs.microsoft.com/en-us/azure/aks/ingress-own-tls
