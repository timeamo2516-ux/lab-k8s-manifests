# Tech Store - Despliegue de la aplicación en Kubernetes

Este paquete contiene los manifiestos Kubernetes para desplegar la aplicación Tech Store en un clúster EKS previamente preparado.

> **Importante:** la creación y configuración del clúster EKS, los add-ons, los permisos IAM, ECR y el AWS Load Balancer Controller se documentan en `0.1_README-aws-setup.md` & `03_README-aws-load-balancer.md`. Este documento se enfoca únicamente en la aplicación.

## 1. Componentes de la aplicación

- Angular 19 + Nginx
- Inventory Service: Java 21 + Spring Boot
- Sales Service: Java 21 + Spring Boot
- PostgreSQL 17
- Namespace dedicado
- ConfigMap y Secret
- StatefulSet con almacenamiento persistente
- Deployments y Services internos
- Ingress para exponer el frontend mediante un ALB ya habilitado en AWS

## 2. Requisitos previos

Antes de continuar, verifica que:

- Tienes acceso al clúster con `kubectl`.
- El contexto de `kubectl` apunta al clúster correcto.
- Las imágenes de la aplicación están publicadas en ECR.
- El AWS Load Balancer Controller está operativo.
- Existe una `StorageClass` compatible con EBS, por ejemplo `gp3`.

Validaciones:

```bash
kubectl config current-context
kubectl get nodes
kubectl get storageclass
kubectl get deployment -n kube-system aws-load-balancer-controller
```

## 3. Configurar las imágenes

Actualiza las propiedades `image:` en los siguientes archivos:

```text
k8s/10-inventory-deployment.yaml
k8s/20-sales-deployment.yaml
k8s/30-web-deployment.yaml
```

Ejemplo:

```yaml
image: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/inventory-service:1.1.0
```

Puedes definir la URI del registro así:

```bash
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export REGISTRY_URI="${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com"
```

Asegúrate de que los nombres y tags de las imágenes coincidan con los publicados en ECR.

## 4. Configurar el Secret de la aplicación

El archivo `k8s/02-secret.yaml` contiene una contraseña de ejemplo para PostgreSQL.

Modifica el valor antes del despliegue:

```yaml
POSTGRES_PASSWORD: change-me-before-production
```

> Este Secret está pensado para el laboratorio. No utilices credenciales en texto plano en un entorno productivo.

## 5. Desplegar la aplicación

Desde la raíz del paquete, ejecuta:

```bash
kubectl apply -k k8s/
```

Verifica los recursos:

```bash
kubectl get all -n tech-store
kubectl get pvc -n tech-store
kubectl get pods -n tech-store -o wide
kubectl get ingress -n tech-store
```

## 6. Esperar a PostgreSQL

```bash
kubectl rollout status statefulset/postgres -n tech-store
kubectl get pvc -n tech-store
```

El PVC debe aprovisionar un volumen persistente mediante la `StorageClass` configurada en el clúster.

## 7. Esperar a los Deployments

```bash
kubectl rollout status deployment/inventory-service -n tech-store
kubectl rollout status deployment/sales-service -n tech-store
kubectl rollout status deployment/web-app -n tech-store
```

## 8. Obtener la URL de la aplicación

```bash
kubectl get ingress tech-store -n tech-store
```

Espera a que el campo `ADDRESS` muestre el nombre del Application Load Balancer.

El ALB puede tardar unos minutos en crearse.

## 9. Validar la aplicación

Abre la dirección del ALB en el navegador. Deberías visualizar el frontend Angular.

El flujo de compra es:

```text
Angular
   |
   v
Sales Service
   |
   v
Inventory Service
   |
   v
PostgreSQL
```

## 10. Pruebas internas

Consultar productos:

```bash
kubectl run curl --rm -it --restart=Never   --image=curlimages/curl   -n tech-store --   curl http://inventory-service:8081/api/products
```

Consultar ventas:

```bash
kubectl run curl --rm -it --restart=Never   --image=curlimages/curl   -n tech-store --   curl http://sales-service:8082/api/sales
```

Crear un producto:

```bash
kubectl run curl --rm -it --restart=Never   --image=curlimages/curl   -n tech-store --   curl -X POST http://inventory-service:8081/api/products   -H 'Content-Type: application/json'   -d '{"name":"Laptop Lenovo","category":"Laptops","price":2499.90,"stock":10}'
```

Probar una compra de dos unidades:

```bash
kubectl run curl --rm -it --restart=Never   --image=curlimages/curl   -n tech-store --   curl -X POST http://sales-service:8082/api/sales   -H 'Content-Type: application/json'   -d '{"productId":1,"quantity":2}'
```

Consultar el stock:

```bash
kubectl run curl --rm -it --restart=Never   --image=curlimages/curl   -n tech-store --   curl http://inventory-service:8081/api/products/1
```

## 11. Logs

```bash
kubectl logs -n tech-store deployment/inventory-service
kubectl logs -n tech-store deployment/sales-service
kubectl logs -n tech-store deployment/web-app
kubectl logs -n tech-store statefulset/postgres
```

## 12. Escalar componentes stateless

```bash
kubectl scale deployment/inventory-service -n tech-store --replicas=3
kubectl scale deployment/sales-service -n tech-store --replicas=3
kubectl scale deployment/web-app -n tech-store --replicas=3
```

PostgreSQL se mantiene con una réplica en este laboratorio.

## 13. Actualizar una aplicación

```bash
kubectl set image deployment/inventory-service   inventory-service="${REGISTRY_URI}/inventory-service:1.2.0"   -n tech-store

kubectl rollout status deployment/inventory-service -n tech-store
```

Rollback:

```bash
kubectl rollout undo deployment/inventory-service -n tech-store
```

## 14. Eliminar la aplicación

```bash
kubectl delete namespace tech-store
```

Esto elimina los recursos del namespace y el PVC de PostgreSQL.

Para eliminar la infraestructura AWS, utiliza el procedimiento documentado por separado en `README-aws-setup.md`.

## 15. Manifiestos incluidos

| Archivo | Recurso | Propósito |
|---|---|---|
| `00-namespace.yaml` | Namespace | Aislar el laboratorio |
| `01-configmap.yaml` | ConfigMap | Configuración no sensible |
| `02-secret.yaml` | Secret | Credenciales de PostgreSQL |
| `03-postgres-statefulset.yaml` | StatefulSet + PVC | PostgreSQL con persistencia |
| `04-postgres-service.yaml` | Service | Acceso interno a PostgreSQL |
| `10-inventory-deployment.yaml` | Deployment | Inventory Service |
| `11-inventory-service.yaml` | Service | Acceso interno al inventario |
| `20-sales-deployment.yaml` | Deployment | Sales Service |
| `21-sales-service.yaml` | Service | Acceso interno a ventas |
| `30-web-deployment.yaml` | Deployment | Angular/Nginx |
| `31-web-service.yaml` | Service | Acceso interno al frontend |
| `40-ingress.yaml` | Ingress | Exposición mediante AWS ALB |

## 16. Consideraciones del laboratorio

- PostgreSQL se ejecuta dentro de EKS únicamente con fines de laboratorio.
- Los microservicios y PostgreSQL se exponen internamente mediante `ClusterIP`.
- El frontend se expone mediante el Ingress.
- Los manifiestos incluyen probes, requests/limits y `securityContext`.
- Para producción se recomienda evaluar TLS, WAF, gestión externa de secretos, NetworkPolicies, HPA, observabilidad y Amazon RDS/Aurora PostgreSQL.
