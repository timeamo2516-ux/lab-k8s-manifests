# Laboratorio: Despliegue de Aplicación Completa en Amazon EKS

Este repositorio contiene los manifiestos de Kubernetes necesarios para desplegar la aplicación del laboratorio (Inventory, Sales y Web App) en un clúster de Amazon EKS.

## Arquitectura de los Manifiestos
Se ha estructurado el despliegue con los siguientes componentes:
- **Namespace:** Entorno aislado (`lab-app`) para todos los recursos.
- **ConfigMap & Secrets:** Externalización de configuraciones y credenciales para la base de datos de manera centralizada.
- **StatefulSet & Volumenes (PVC):** Para la base de datos PostgreSQL garantizando la persistencia de los datos.
- **Deployments:** Para los microservicios de Spring Boot (`inventory-service` y `sales-service`) y el frontend en Angular (`web-app`).
- **Services:** Para la comunicación interna dentro del clúster.
- **Ingress:** Para exponer el frontend y las APIs al exterior (requiere un Ingress Controller como NGINX o AWS ALB).

## Requisitos Previos
1. **AWS CLI** configurado.
2. **kubectl** instalado y configurado para conectarse al clúster EKS (`aws eks update-kubeconfig --name <nombre-cluster> --region <region>`).
3. Clúster EKS activo con nodos disponibles.
4. **Ingress Controller** instalado en el clúster (por defecto, los manifiestos asumen `ingress-nginx`).
5. Amazon ECR o Docker Hub listo para recibir las imágenes.

## Paso 1: Construir y Subir Imágenes (Push)
Antes de aplicar los YAMLs, debes construir las imágenes a partir del código fuente proporcionado en el `.zip` original y subirlas a un Container Registry.

```bash
# Ejemplo de build y push para Docker Hub o ECR
docker build -t <TU_REGISTRY>/inventory-service:latest ./inventory-service
docker build -t <TU_REGISTRY>/sales-service:latest ./sales-service
docker build -t <TU_REGISTRY>/web-app:latest ./web-app

docker push <TU_REGISTRY>/inventory-service:latest
docker push <TU_REGISTRY>/sales-service:latest
docker push <TU_REGISTRY>/web-app:latest
```

## Paso 2: Actualizar las Imágenes en los Manifiestos
Abre los archivos `k8s/04-inventory-service.yaml`, `k8s/05-sales-service.yaml` y `k8s/06-web-app.yaml` y reemplaza `<TU_REGISTRY>` por la ruta real de tus imágenes.

## Paso 3: Desplegar la Aplicación en EKS
Aplica los archivos en orden:

```bash
# 1. Crear el Namespace
kubectl apply -f k8s/00-namespace.yaml

# 2. Desplegar Configuraciones y Secretos
kubectl apply -f k8s/01-configmap.yaml
kubectl apply -f k8s/02-secrets.yaml

# 3. Desplegar la Base de Datos (StatefulSet)
kubectl apply -f k8s/03-database.yaml

# 4. Desplegar los Microservicios
kubectl apply -f k8s/04-inventory-service.yaml
kubectl apply -f k8s/05-sales-service.yaml

# 5. Desplegar el Frontend
kubectl apply -f k8s/06-web-app.yaml

# 6. Desplegar el Ingress
kubectl apply -f k8s/07-ingress.yaml
```

## Paso 4: Verificación
Verifica que los pods estén corriendo:
```bash
kubectl get all -n lab-app
```

Obtén la URL o el External IP del Ingress para acceder a la aplicación:
```bash
kubectl get ingress -n lab-app
```
*Nota: Si estás usando AWS y el Ingress NGINX fue expuesto mediante un LoadBalancer, puede tardar unos minutos en provisionar el clásico ELB o NLB.*
