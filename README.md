# final-project-pilarAlonsoSTEMDO

![Sin título-2024-06-13-0739](https://github.com/stemdo-labs/final-project-pilarAlonsoSTEMDO/assets/166375061/6535855c-6853-4b5a-9c53-9d1bcd263bca)

# Terraform

Dentro del proyecto de Terraform tendremos:

- Una **Virtual Network (VNet)** que contiene todas las subredes.

## Subredes para:

- **AKS Cluster** con un **Load Balancer** asociado.
- **DB VM**.
- **Jump Host**.

## Componentes:

- **Load Balancer**: para distribuir el tráfico entrante al AKS Cluster.
- **Public IP**: asociada al Jump Host.

## Acceso a la App


[Usuario] --> [Public IP] --> [Load Balancer] --> [AKS Cluster] --> [Aplicación]

                         ┌─────────────────────────────┐
                         │       Azure Kubernetes      │
                         │         Service (AKS)       │
                         │                             │
                         │  ┌───────────────────────┐  │
                         │  │   App Deployment      │  │
                         │  └───────────────────────┘  │
                         │      (2 replicas)          │
                         │         │                 │
                         │         │                 │
                         │  ┌───────────────────────┐  │
                         │  │    Load Balancer      │  │
                         │  └───────────────────────┘  │
                         │                             │
                         │         │                  │
                         │         │                  │
                         └─────────┼──────────────────┘
                                   │
                                   │
                 ┌─────────────────┴─────────────────┐
                 │                                   │
      ┌──────────▼──────────┐            ┌───────────▼───────────┐
      │     VM for DB       │            │     VM for Backups    │
      │   (MySQL/PostgreSQL)│            │                      │
      └─────────────────────┘            └───────────────────────┘

    ┌─────────────────────────────────────────────────────────────────┐
    │                         Virtual Network (VNet)                  │
    │ ┌───────────────────────────────────────────────────────────┐  │
    │ │                  Subnet for AKS Nodes                     │  │
    │ └───────────────────────────────────────────────────────────┘  │
    │ ┌───────────────────────────────────────────────────────────┐  │
    │ │          Subnet for VMs (DB, Backups)                     │  │
    │ └───────────────────────────────────────────────────────────┘  │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘


```plaintext
## Estructura

terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── provider.tf
├── modules/
│   ├── network/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   ├── nsg/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   ├── load_balancer/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   ├── aks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   ├── vm/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
└── terraform.tfvars
```




## Configuring Azure Container Registry (ACR) and Azure Kubernetes Service (AKS)

This section details the steps required to configure an Azure Container Registry (ACR) and an Azure Kubernetes Service (AKS) cluster in Azure. It also includes instructions on creating and using Docker secrets in your AKS cluster and deleting virtual machine disks in Azure.

### Steps to Configure ACR and AKS

1. **Obtain the ACR ID**
    ```sh
    ACR_ID=$(az acr show --name palonsoACR --resource-group rg-palonso-dvfinlab --query "id" --output tsv)
    echo "ACR_ID obtained: $ACR_ID"
    ```

2. **Obtain the AKS Cluster Identity ID**
    ```sh
    AKS_IDENTITY_ID=$(az aks show --resource-group rg-palonso-dvfinlab --name aks-cluster --query "identityProfile.kubeletidentity.objectId" --output tsv)
    echo "AKS_IDENTITY_ID obtained: $AKS_IDENTITY_ID"
    ```

3. **Assign the AcrPull Role to the AKS Cluster**
    ```sh
    az role assignment create --assignee $AKS_IDENTITY_ID --role AcrPull --scope $ACR_ID
    echo "AcrPull role assigned to the AKS cluster."
    ```

4. **Configure kubectl for the AKS Cluster**
    ```sh
    az aks get-credentials --resource-group rg-palonso-dvfinlab --name aks-cluster
    echo "kubectl configured for the AKS cluster."
    ```
  ###Como en nuestro caso no tenemos la autorización correspondiente consumiremos el secreto de docker con las credenciales facilitadas en el ACR

5. **Create Docker Secret in AKS Cluster**
    ```sh
    kubectl create secret docker-registry regcred \
      --docker-server=palonsoacr.azurecr.io \
      --docker-username=palonsoACR \
      --docker-password=<YOUR_PASSWORD> \
      --docker-email=palonso@stemdo.io
    echo "Docker Secret created in AKS cluster."
    ```

    Replace `<YOUR_PASSWORD>` with your actual ACR password. This secret will be used to authenticate Docker with the ACR from the AKS cluster.

### Deleting Virtual Machine Disks in Azure

To delete the disks of virtual machines in Azure, use the following commands:

```sh
az disk delete --resource-group RG-PALONSO-DVFINLAB --name backup-vm-osdisk --yes
az disk delete --resource-group RG-PALONSO-DVFINLAB --name db-vm-osdisk --yes
```
5. **Acceso a la app a través del clúster**

![image](https://github.com/stemdo-labs/final-project-pilarAlonsoSTEMDO/assets/166375061/af7545a7-95ca-4e76-84f7-c11821e5c555)
5. **Conexión por ssh a la vm_backup a través de su IP pública **
```sh
ssh -i ~/.ssh/id_rsa adminuser@52.174.32.157
```


![image](https://github.com/stemdo-labs/final-project-pilarAlonsoSTEMDO/assets/166375061/692e2965-731f-42e1-924a-6c8e58cccef4)


