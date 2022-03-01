Example Voting App
=========
Run Vagrant 

cd vagrant-kubernetes 

Review Readme file for how to run k8s using vagrant with pre-installed vagrant and virtualbox and sample deployment



Run the app in Kubernetes
-------------------------

The folder k8s-specifications contains the yaml specifications of the Voting App's services.

Create AKS (azure k8s cluster) and ACR (azure container registery) get credentials 
1. Open up your AKS cluster in the azure portal (portal.azure.com)
2. Create aks cluster with access perimission to acr and login to azure with az login
3. get access keys from acr ( ACR_NAME, ACR_PASSWORD, ACR_USERNAME) to github secrets in the repo
4. we need to allow remote access to aks so we need to create the service principal
    a. az ad sp create-for-rbac \
    --name upgrade-test \
    --skip-assignment
    b. Create a secret called SERVICE_PRINCIPAL_APP_ID and add the az ad sp create-for-rbac output value appId
    c. Create a secret called SERVICE_PRINCIPAL_SECRET and add the az ad sp create-for-rbac output value password
    d. Create a secret called SERVICE_PRINCIPAL_TENANT and add the az ad sp create-for-rbac output value tenant
    e. Create a secret named CLUSTER_RESOURCE_GROUP_NAME with the name of the resource group of the AKS cluster created above ($RG)
    f. Create a secret named CLUSTER_NAME with the name of the AKS cluster created above ($CLUSTER)
    g. Create a secret named ACR_NAME with the name of the container registry ($ACR)
Now we need to give this service principal a few more permissions. First, grant it the ability to push an image to the container registry:
 az role assignment create \
    --role AcrPush \
    --assignee-principal-type ServicePrincipal \
    --assignee-object-id $(az ad sp show \
        --id $SERVICE_PRINCIPAL_APP_ID \
        --query objectId -o tsv) \
    --scope $(az acr show --name $ACR --query id -o tsv)

That command grants the service principal the AcrPush role for the container registry that was created above.

Grant the service principal the ability to retrieve the credentials of the AKS cluster (az aks get-credentials):

$ az role assignment create \
    --role "Azure Kubernetes Service Cluster User Role" \
    --assignee-principal-type ServicePrincipal \
    --assignee-object-id $(az ad sp show \
        --id $SERVICE_PRINCIPAL_APP_ID \
        --query objectId -o tsv) \
    --scope $(az aks show \
        --resource-group $RG \
        --name $CLUSTER \
        --query id -o tsv)


we want to grant this service principal the ability to read and write to the default namespace. 
az role assignment create \
    --role "Azure Kubernetes Service RBAC Writer" \
    --assignee-principal-type ServicePrincipal \
    --assignee-object-id $(az ad sp show \
        --id $SERVICE_PRINCIPAL_APP_ID \
        --query objectId -o tsv) \
    --scope "$(az aks show \
        --resource-group $RG \
        --name $CLUSTER \
        --query id -o tsv)/namespaces/default"


after collecting all of the info above add all needed to secrets ( CLUSTER_NAME, CLUSTER_RESOURCE_GROUP_NAME, SERVICE_PRINCIPAL_APP_ID, SERVICE_PRINCIPAL_SECRET, SERVICE_PRINCIPAL_TENANT)


First create the vote namespace

```
For database: add secrets USER, DATABASE, PASSWORD  in base64 encode
```
$ kubectl create namespace vote
```

Run the following command to create the deployments and services objects:
```
$ kubectl create -f k8s-specifications/
deployment "db" created
service "db" created
deployment "redis" created
service "redis" created
deployment "result" created
service "result" created
deployment "vote" created
service "vote" created
deployment "worker" created
```

The vote interface is then available on port 31000 on each host of the cluster, the result one is available on port 31001.

In Case of Vagrant
-----
Create secret with ACR creds to access the container register 
Replace loadbalancer with nodeport and use default up allocated in vagrant 10.0.0.10 to access services in case of aks loadbalancer, for production cluster ip is recommended.
