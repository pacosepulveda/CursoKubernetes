# Ejercicios Módulo 10 - Helm

## Objetivo

En este módulo vamos a practicar Helm 3 sobre el clúster Kubernetes del laboratorio creado con `kubeadm` en EC2.

Evitaremos el repositorio `stable`, ya obsoleto para prácticas actuales, y trabajaremos con dos enfoques:

1. Instalar un chart público de Bitnami.
2. Crear e instalar un chart local sencillo.

---

## 1. Instalar Helm 3

Ejecuta estos comandos en el nodo desde el que usas `kubectl`, normalmente el nodo control-plane:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Comprueba la instalación:

```bash
helm version --short
```

Opcionalmente, activa autocompletado de Bash:

```bash
sudo apt-get update
sudo apt-get install -y bash-completion
helm completion bash | sudo tee /etc/bash_completion.d/helm >/dev/null
source /etc/bash_completion
```

---

## 2. Añadir repositorios de charts

Añadimos el repositorio de Bitnami y actualizamos el índice local:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Buscar charts relacionados con nginx:

```bash
helm search repo nginx
helm search repo bitnami/nginx
```

Podemos inspeccionar los valores configurables del chart:

```bash
helm show values bitnami/nginx | less
```

---

## 3. Instalar nginx con Helm usando NodePort

Creamos un namespace para el módulo:

```bash
kubectl create namespace helm-demo
```

Instalamos el chart. Forzamos `service.type=NodePort` para que funcione en el clúster kubeadm del laboratorio:

```bash
helm install mywebserver bitnami/nginx \
  --namespace helm-demo \
  --set service.type=NodePort \
  --set service.nodePorts.http=30080
```

Comprobamos la release de Helm:

```bash
helm list -n helm-demo
helm status mywebserver -n helm-demo
```

Comprobamos los objetos creados:

```bash
kubectl get deploy,pod,svc -n helm-demo
```

Acceso desde dentro del clúster:

```bash
kubectl run curl-test -n helm-demo --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- \
  curl -I http://mywebserver-nginx.helm-demo.svc.cluster.local
```

Acceso mediante NodePort desde una máquina que pueda llegar a la IP privada o pública de un nodo:

```bash
kubectl get svc -n helm-demo mywebserver-nginx
kubectl get nodes -o wide
```

Usa la IP de un nodo y el puerto `30080`:

```bash
curl http://<IP_DEL_NODO>:30080
```

En AWS, el Security Group debe permitir el puerto TCP `30080` desde tu IP o desde el rango que uses en el laboratorio.

---

## 4. Actualizar una release

Cambiamos el número de réplicas:

```bash
helm upgrade mywebserver bitnami/nginx \
  --namespace helm-demo \
  --reuse-values \
  --set replicaCount=2
```

Verificamos:

```bash
helm history mywebserver -n helm-demo
kubectl get deploy,pod -n helm-demo
```

---

## 5. Rollback de una release

Consulta el historial:

```bash
helm history mywebserver -n helm-demo
```

Vuelve a la revisión anterior. Sustituye `1` por la revisión que quieras restaurar:

```bash
helm rollback mywebserver 1 -n helm-demo
```

Comprueba el estado:

```bash
helm status mywebserver -n helm-demo
kubectl get deploy,pod -n helm-demo
```

---

## 6. Desinstalar una release

```bash
helm uninstall mywebserver -n helm-demo
```

Comprueba que desaparecieron los objetos gestionados por la release:

```bash
helm list -n helm-demo
kubectl get all -n helm-demo
```

---

# Ejercicio 2: Crear e instalar un chart local

En este ejercicio usaremos el chart incluido con el módulo en la carpeta `modulo10-chart`.

La estructura principal es:

```text
modulo10-chart/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-prod.yaml
└── templates/
    ├── _helpers.tpl
    ├── configmap.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── NOTES.txt
```

Este chart despliega:

- Un `ConfigMap` con una página HTML.
- Un `Deployment` con nginx.
- Un `Service` de tipo `NodePort`.

---

## 7. Validar el chart local

Desde el directorio donde tengas los archivos del módulo:

```bash
helm lint ./modulo10-chart
```

Renderizar los manifiestos sin instalarlos:

```bash
helm template web-local ./modulo10-chart --namespace helm-demo
```

Simular la instalación contra el API Server:

```bash
helm install web-local ./modulo10-chart \
  --namespace helm-demo \
  --dry-run \
  --debug
```

---

## 8. Instalar el chart local

```bash
helm install web-local ./modulo10-chart --namespace helm-demo
```

Comprobar objetos:

```bash
helm list -n helm-demo
kubectl get deploy,pod,svc,cm -n helm-demo -l app.kubernetes.io/instance=web-local
```

Probar acceso interno:

```bash
kubectl run curl-local -n helm-demo --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- \
  curl http://web-local-modulo10-web.helm-demo.svc.cluster.local
```

Probar acceso NodePort:

```bash
kubectl get svc -n helm-demo web-local-modulo10-web
curl http://<IP_DEL_NODO>:30090
```

---

## 9. Cambiar valores con `--set`

Actualiza el contenido HTML y el número de réplicas:

```bash
helm upgrade web-local ./modulo10-chart \
  --namespace helm-demo \
  --set replicaCount=3 \
  --set html.title="Helm upgrade" \
  --set html.body="Contenido actualizado con helm upgrade"
```

Verifica:

```bash
helm history web-local -n helm-demo
kubectl get deploy,pod -n helm-demo -l app.kubernetes.io/instance=web-local
curl http://<IP_DEL_NODO>:30090
```

---

## 10. Usar archivos de valores

Instalar una release de desarrollo:

```bash
helm install web-dev ./modulo10-chart \
  --namespace helm-demo \
  -f ./modulo10-chart/values-dev.yaml
```

Instalar una release de producción:

```bash
helm install web-prod ./modulo10-chart \
  --namespace helm-demo \
  -f ./modulo10-chart/values-prod.yaml
```

Comprobar que son releases independientes:

```bash
helm list -n helm-demo
kubectl get svc -n helm-demo
```

Los NodePorts configurados son:

- `web-local`: `30090`
- `web-dev`: `30091`
- `web-prod`: `30092`

---

## 11. Rollback del chart local

Consulta el historial:

```bash
helm history web-local -n helm-demo
```

Restaura una revisión anterior:

```bash
helm rollback web-local 1 -n helm-demo
```

Comprueba:

```bash
helm status web-local -n helm-demo
curl http://<IP_DEL_NODO>:30090
```

---

## 12. Limpieza

Eliminar releases:

```bash
helm uninstall web-local -n helm-demo
helm uninstall web-dev -n helm-demo
helm uninstall web-prod -n helm-demo
```

Eliminar namespace:

```bash
kubectl delete namespace helm-demo
```

---
- Para este laboratorio es más fiable usar `NodePort` y abrir explícitamente los puertos necesarios en el Security Group.
- El ejercicio opcional original basado en App Mesh/EKS se ha eliminado porque depende de EKS, Elastic Load Balancer, App Mesh y recursos AWS que no forman parte del clúster kubeadm del curso.
