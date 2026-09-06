# Script to update an AKS cluster to enable OIDC issuer and workload identity.
```shell
export CLUSTER_RG="rg-harness-sandbox-local"
export CLUSTER_NAME="aks-harness-delegate-free"
export LOCATION="eastus"
export IDENTITY_RG="rg-harness-sandbox-local"
export SUBSCRIPTION_ID="ebc9c4d2-xxxx-48f0-xxx-593dfexxxxxx"
export K8S_NAMESPACE="tf-lab"
export K8S_SERVICE_ACCOUNT="tf-runner-sa"
export TENANT_ID=$(az account show --query tenantId -o tsv)


# [optional] Update the AKS cluster to enable OIDC issuer and workload identity, if not already enabled.
az aks update \
  --resource-group $CLUSTER_RG \
  --name $CLUSTER_NAME \
  --enable-oidc-issuer \
  --enable-workload-identity


export AKS_OIDC_ISSUER=$(az aks show \
  --resource-group $CLUSTER_RG \
  --name $CLUSTER_NAME \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)
echo "AKS OIDC Issuer URL: $AKS_OIDC_ISSUER"
```




# Create Identity
``` shell
az identity create \
  --name tf-runner-identity \
  --resource-group $IDENTITY_RG \
  --location $LOCATION

export UMI_CLIENT_ID=$(az identity show \
  --name tf-runner-identity \
  --resource-group $IDENTITY_RG \
  --query clientId -o tsv)

echo "----------------------------------------"
echo "UMI Client ID: $UMI_CLIENT_ID"
echo "----------------------------------------"

# Assign Contributor (or required scope) to the Subscription
az role assignment create \
  --assignee $UMI_CLIENT_ID \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"

az identity federated-credential create \
  --name "tf-runner-federated-cred" \
  --identity-name tf-runner-identity \
  --resource-group $IDENTITY_RG \
  --issuer "$AKS_OIDC_ISSUER" \
  --subject "system:serviceaccount:$K8S_NAMESPACE:$K8S_SERVICE_ACCOUNT" \
  --audience "api://AzureADTokenExchange"

```

# Create Kubernetes manifests for workload identity binding

``` shell
cat <<EOF > tf-all-in-one.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: $K8S_NAMESPACE
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tf-runner-sa
  namespace: tf-lab
  annotations:
    azure.workload.identity/client-id: "$UMI_CLIENT_ID"
---
apiVersion: v1
kind: Pod
metadata:
  name: tf-runner-pod
  namespace: tf-lab
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: tf-runner-sa
  containers:
    - name: tf-cli
      image: hashicorp/terraform:light
      command: ["sleep", "3600"]
      workingDir: /workspace
      env:
        # Terraform Auth Controls
        - name: ARM_USE_OIDC
          value: "true"
        - name: ARM_OIDC_TOKEN_FILE_PATH
          value: "/var/run/secrets/azure/tokens/azure-identity-token"
        - name: ARM_SUBSCRIPTION_ID
          value: "$SUBSCRIPTION_ID"
        
        # Explicit ARM Mappings for Provider Authorizer
        - name: ARM_CLIENT_ID
          value: "$UMI_CLIENT_ID"
        - name: ARM_TENANT_ID
          value: "$TENANT_ID"
EOF

kubectl exec -it tf-runner-pod -- sh
cd /tmp

cat <<EOF > main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
  use_oidc = true
}

resource "azurerm_resource_group" "example" {
  name     = "rg-tf-workload-identity-demo"
  location = "eastus"
}

EOF
```
# Apply the k8s manifest
``` kubectl
kubectl apply -f tf-all-in-one


kubectl exec -it tf-runner-pod -- sh
cd /tmp

cat <<EOF > main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
  use_oidc = true
}

resource "azurerm_resource_group" "example" {
  name     = "rg-tf-workload-identity-demo"
  location = "eastus"
}
EOF
```

# Terraform
```
terraform init
terraform plan
terraform apply --auto-approve
```

