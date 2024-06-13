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

```plaintext
[Usuario] --> [Public IP] --> [Load Balancer] --> [AKS Cluster] --> [Aplicación]


