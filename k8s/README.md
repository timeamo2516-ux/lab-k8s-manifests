# Manifiestos de la aplicación Tech Store

Esta carpeta contiene **únicamente los manifiestos Kubernetes de la aplicación**. La creación y configuración de la infraestructura AWS se encuentra en `0.1_README-aws-setup.md` & `03_README-aws-load-balancer.md`.

## Orden recomendado de lectura

1. `00-namespace.yaml`
2. `01-configmap.yaml`
3. `02-secret.yaml`
4. `03-postgres-statefulset.yaml`
5. `04-postgres-service.yaml`
6. `10-inventory-deployment.yaml`
7. `11-inventory-service.yaml`
8. `20-sales-deployment.yaml`
9. `21-sales-service.yaml`
10. `30-web-deployment.yaml`
11. `31-web-service.yaml`
12. `40-ingress.yaml`
13. `kustomization.yaml`

## Desplegar la aplicación

Desde la raíz del proyecto:

```bash
kubectl apply -k k8s/
```

## Validar el despliegue

```bash
kubectl get all -n tech-store
kubectl get pvc -n tech-store
kubectl get ingress -n tech-store
```

## Eliminar despliegue
```bash
kubectl delete namespace tech-store
```

## Alcance de esta carpeta

Aquí se configura solamente:

- Namespace de la aplicación.
- Configuración no sensible.
- Secret de laboratorio.
- PostgreSQL y su almacenamiento persistente.
- Deployments de los microservicios y del frontend.
- Services internos.
- Ingress de la aplicación.

No se crean desde estos manifiestos:

- El clúster EKS.
- Los grupos de nodos.
- Los add-ons de EKS.
- Los roles o políticas IAM.
- Los repositorios ECR.
- El AWS Load Balancer Controller.
- La infraestructura de red de AWS.
