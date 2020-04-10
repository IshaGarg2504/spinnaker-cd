#@ spinnaker-cd

## Prerequisites
- Azure Account
- AKS Cluster setup

## Install with Halyard

https://www.spinnaker.io/setup/install/halyard/#install-on-debianubuntu-and-macos

```
mkdir -p ./spin_hal
cd spin_hal
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
sudo bash InstallHalyard.sh
hal -v
```
## Add K&S account - Applciations / Services Cluster

https://www.spinnaker.io/setup/install/providers/kubernetes-v2/

Enable kubernetes

```
hal config provider kubernetes enable
```
Add K8S Account

```
CONTEXT=$(kubectl config current-context)
hal config provider kubernetes account add <ACCOUNT_NAME> --provider-version v2 --context $CONTEXT
```

## Environemnt for Spinnaker

https://www.spinnaker.io/setup/install/environment/#distributed-installation

Deploy Spinnaker on common devops cluster or a seperate cluster.

```
hal config deploy edit --type distributed --account-name <ACCOUNT_NAME>
```

## Configure Storage - Azure Storage

- Create Azure Storage Account and fetch the key

```
az storage account create --resource-group <RESOURCE_GROUP_NAME> --sku STANDARD_LRS --name <STORAGE_ACCOUNT>
STORAGE_ACCOUNT_KEY=$(az storage account keys list --resource-group <RESOURCE_GROUP_NAME> --account-name <STORAGE_ACCOUNT> --query [0].value | tr -d '"')
hal config storage azs edit \
        --storage-account-name <STORAGE_ACCOUNT> \
        --storage-account-key $STORAGE_ACCOUNT_KEY
# Set the storage as AZS
hal config storage edit --type azs
```

## Add docker registry accounts

Repeat this for every registry with repositories. Used ACR for this example

Create ACR(Azure Container Registry)
```
az acr create --resource-group <RESOURCE_GROUP_NAME> --name <REG_NAME> --sku Basic
```
```
REG_ADDRESS=<>
REPOSITORIES=jenkins hello-world
USERNAME="REG_NAME"
PASSWORD="<PASSWORD/Key>" //Use this to get password "az acr credential show -n <REG_NAME>"
hal config provider docker-registry account add <REG_ACCOUNT> --address $REG_ADDRESS --username $USERNAME --password $PASSWORD --repositories $REPOSITORIES
# ADD Kubernetes Secrets
kubectl create secret docker-registry <REG_NAME> --docker-server=$REG_ADDRESS --docker-username=$USERNAME --docker-password=$PASSWORD
```

## Deploy Spinnaker

```
hal version list
# Pick the latest version.
VERSION=<>
hal config version edit --version $VERSION

# Deploy
hal deploy apply
```

## Test & Use Spinnaker

Edit/override the API and UI urls to access spinnaker

```
hal config security ui edit --override-base-url "http://<CLUSTERIP/K8S_NODE_IP/LB>:9000"
hal config security api edit --override-base-url "http://<CLUSTERIP/K8S_NODE_IP/LB>:8084"
open http://<CLUSTERIP/K8S_NODE_IP/LB>:9000
```
# Spinnaker Usage

## Create Application

- Provide application name, email and submit

## Create Pipeline

Create pipelines within an application

- Give it a name and submit
- Pipeline Config
  - Add "Expected Artifacts" -> Account: "docker-registry" -> Docker image: "<REPO-URL>" -> Display name: "<ARTIFACT_NAME>"
  - Add "Automated Triggers" -> Type: "docker-registry" -> Registry Name: "<REG_ACCOUNT>" -> Image: "<REPO>"
- Add a new stage
  - Type = Deploy (manifest)
  - Name = Deploy QA
  - Account = <K8S_ACCOUNT_NAME>
  - Namespace = spinnaker
  - Manifest source = text
  - Paste in the content from `k8s-manifest/manifests.yml` for the manifest in the text area  
  - Required Artifacts to Bind = <ARTIFACT_NAME>
  - Save
- Run pipeline and verify the pods on `service cluster`

```
kubectl get pods -n spinnaker
```

Create a service within the application

- Go to Clusters
- Create New Server Group
- Paste in the service manifest for website. Check the one from `k8s-manifest/service.yml`
- Hit `Create` which should apply that manifest into the service cluster
- Verify the service on `service cluster` and test

```
kubectl get svc -n spinnaker
