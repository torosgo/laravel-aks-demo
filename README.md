# Laravel on AKS (Azure Kubernetes Service) Demo Application

Learn how to deploy a Laravel (PHP) application to AKS with load balancer and scalibility features.

## Installation Updates

### Create AKS Cluster

```bash
REGION_NAME=centralus
RESOURCE_GROUP=laraks-rg
SUBNET_NAME=aks-subnet
VNET_NAME=aks-vnet

az group create \
    --name $RESOURCE_GROUP \
    --location $REGION_NAME

az network vnet create \
    --resource-group $RESOURCE_GROUP \
    --location $REGION_NAME \
    --name $VNET_NAME \
    --address-prefixes 10.0.0.0/8 \
    --subnet-name $SUBNET_NAME \
    --subnet-prefix 10.240.0.0/16

SUBNET_ID=$(az network vnet subnet show \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $VNET_NAME \
    --name $SUBNET_NAME \
    --query id -o tsv)

VERSION=$(az aks get-versions \
    --location $REGION_NAME \
    --query 'orchestrators[?!isPreview] | [-1].orchestratorVersion' \
    --output tsv)

AKS_CLUSTER_NAME=aksworkshop-$RANDOM

az aks create \
    --resource-group $RESOURCE_GROUP \
    --name $AKS_CLUSTER_NAME \
    --vm-set-type VirtualMachineScaleSets \
    --enable-cluster-autoscaler \
    --min-count 1 \
    --max-count 5 \
    --node-count 1 \
    --node-vm-size Standard_DS2_v2 \
    --load-balancer-sku standard \
    --enable-addons monitoring \
    --location $REGION_NAME \
    --kubernetes-version $VERSION \
    --network-plugin azure \
    --vnet-subnet-id $SUBNET_ID \
    --service-cidr 10.2.0.0/24 \
    --dns-service-ip 10.2.0.10 \
    --docker-bridge-address 172.17.0.1/16 \
    --generate-ssh-keys \
    --tags 'project=aksworkshop' \
    --network-policy calico \
    --no-wait

az aks get-credentials \
    --resource-group $RESOURCE_GROUP \
    --name $AKS_CLUSTER_NAME

kubectl get nodes
```

### Create ACR and associate with AKS


```bash
ACR_NAME=acr$RANDOM
echo export ACR_NAME=$ACR_NAME>> .laraksenv

az acr create \
    --resource-group $RESOURCE_GROUP \
    --location $REGION_NAME \
    --name $ACR_NAME \
    --sku Standard

az aks update \
    --name $AKS_CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --attach-acr $ACR_NAME
```

### Copy your Laravel project folder contents or Create new one

```bash
#// If existing project
cd yourexistingprojectfolder
git clone https://github.com/torosgo/laravel-aks-demo.git .

#// If new project
mkdir laraks && cd laraks
git clone https://github.com/torosgo/laravel-aks-demo.git .
docker run --rm --user $(id -u):$(id -g) -v $(pwd):/opt -w /opt composer create-project --prefer-dist laravel/laravel .
```

### Set API_KEY, CONTAINERTAG and CONTAINERREGISTRY

```bash
APP_KEY=$(cat .env|grep APP_KEY|head -1|awk -F "=" '{print $2}')
echo export APP_KEY=$APP_KEY>> .laraksenv

CONTAINERTAG=laraks:v1.12
echo export CONTAINERTAG=$CONTAINERTAG>> .laraksenv

CONTAINERREGISTRY=$ACR_NAME.azurecr.io
echo export CONTAINERREGISTRY=$CONTAINERREGISTRY>> .laraksenv
```

### Optional step: Build and run in your dev environment

```bash
docker build -t $CONTAINERTAG .
docker run -ti -p 8888:80 -e APP_KEY=$APP_KEY $CONTAINERTAG
#// To browse on your localhost enter localhost:8888 in your browser address bar

```

### Build and push docker image in ACR

```bash
az acr build \
    --registry $ACR_NAME \
    --image $CONTAINERTAG .

az acr repository list \
    --name $ACR_NAME \
    --output table
```

### Deploy App


```bash
APPNS=laraks
kubectl create namespace $APPNS
kubectl config set-context --current --namespace $APPNS

#// Editing container registry address in laraks-deployment.yaml before creating the deployment
sed -i -e "s/{{CONTAINERREGISTRY}}/$CONTAINERREGISTRY/g" -e "s/{{CONTAINERTAG}}/$CONTAINERTAG/g" -e "s/{{APP_KEY}}/$APP_KEY/g" laraks-deployment.yaml
kubectl apply -f laraks-deployment.yaml 

// Execute this to see pods scaling
kubectl scale --replicas=3 deployment/laraks
kubectl get deployment
```

### Deploy Service


```bash
kubectl apply -f laraks-svc-pri.yaml 
kubectl get svc
```

### Deploy Ingress


```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo update
kubectl create namespace ingress
helm search repo stable/nginx-ingress
helm install nginx-ingress stable/nginx-ingress \
    --namespace ingress \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux
kubectl get service nginx-ingress-controller --namespace ingress -w

// Editing ingress public ip in laraks-ingress.yaml before creating laraks ingress
LARASVCPRIIP=$(kubectl get service nginx-ingress-controller --namespace ingress|grep nginx-ingress-controller| awk '{print $4}')
echo export LARASVCPRIIP=$LARASVCPRIIP>> .laraksenv
sed -i -e "s/{{LARASVCPRIIP}}/$LARASVCPRIIP/g" laraks-ingress.yaml
kubectl apply -f laraks-ingress.yaml
kubectl get ing
```


## References: 

[AKS Tutorial](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-app)

[Deploying laravel to kubernetes](https://learnk8s.io/blog/deploying-laravel-to-kubernetes/)
 
[Laravel Framework](https://laravel.com/)



## Support

No SLA. Continuous development. Use at your own risk. Please read License

## Contribution

Contributions are welcome.


## Copyright

Copyright &copy; @torosgo 2020.


## License

The Laravel framework and this demo are open source software licensed under the [MIT license](https://opensource.org/licenses/MIT).
