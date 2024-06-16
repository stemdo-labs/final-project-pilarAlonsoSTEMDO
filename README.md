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




## Configurar ACR Y  AKS


### Steps to Configure ACR and AKS

1. **Obtener el ACR ID**
    ```sh
    ACR_ID=$(az acr show --name palonsoACR --resource-group rg-palonso-dvfinlab --query "id" --output tsv)
    echo "ACR_ID obtained: $ACR_ID"
    ```

2. **Obtener el AKS Cluster Identity ID**
    ```sh
    AKS_IDENTITY_ID=$(az aks show --resource-group rg-palonso-dvfinlab --name aks-cluster --query "identityProfile.kubeletidentity.objectId" --output tsv)
    echo "AKS_IDENTITY_ID obtained: $AKS_IDENTITY_ID"
    ```

3. **Asignar el ACRPullRole  al the AKS Cluster**
    ```sh
    az role assignment create --assignee $AKS_IDENTITY_ID --role AcrPull --scope $ACR_ID
    echo "AcrPull role assigned to the AKS cluster."
    ```

4. **Configurar kubectl for the AKS Cluster**
    ```sh
    az aks get-credentials --resource-group rg-palonso-dvfinlab --name aks-cluster
    echo "kubectl configured for the AKS cluster."
    ```
    Sin permisos esta forma de hacerlo nos dará errores por la autorización denegada al no tener un rol que permita crear roles.
0. **Como en nuestro caso no tenemos la autorización correspondiente consumiremos el secreto de docker con las credenciales facilitadas en el ACR**
  

5. **Crear Docker Secret en AKS Cluster**
    ```sh
    kubectl create secret docker-registry regcred \
      --docker-server=palonsoacr.azurecr.io \
      --docker-username=palonsoACR \
      --docker-password=<YOUR_PASSWORD> \
      --docker-email=palonso@stemdo.io
    echo "Docker Secret created in AKS cluster."
    ```

    Replace `<YOUR_PASSWORD>` with your actual ACR password. This secret will be used to authenticate Docker with the ACR from the AKS cluster.

### Borrar discos de las vm

Los discos de almacenamiento de las vms no se borrarán con el destroy

```sh
az disk delete --resource-group RG-PALONSO-DVFINLAB --name backup-vm-osdisk --yes
az disk delete --resource-group RG-PALONSO-DVFINLAB --name db-vm-osdisk --yes
```
5. **Acceso a la app a través del clúster**

![image](https://github.com/stemdo-labs/final-project-pilarAlonsoSTEMDO/assets/166375061/af7545a7-95ca-4e76-84f7-c11821e5c555)
La Ip pública se ha tenido que crear dentro del grupo de recursos Mg donde se localiza el cluster. 

# Ansible
La vm_backup tendrdá una ip pública y funcionará como nodo maestro.

1. **Conexión por ssh a la vm_backup**

```sh
ssh -i ~/.ssh/id_rsa adminuser@52.174.32.157
```

![image](https://github.com/stemdo-labs/final-project-pilarAlonsoSTEMDO/assets/166375061/692e2965-731f-42e1-924a-6c8e58cccef4)

2. **Instalar ansible en  vm_backup**
 ```sh
   sudo apt update
sudo apt install ansible -y
```
2. **Creo el inventario y los playbooks**
   

 Para poder almacenar la copia de la bd en azure tengo que crear un contenedor dentro de mi storage account y usar su key.
```ssh
az storage container create --name mycontainer --account-name stapalonsodvfinlab
``



