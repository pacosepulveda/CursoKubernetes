# Ejercicio guiado: Publicación de aplicaciones con Kubernetes Gateway API y HTTPRoute

## 1. Objetivo del ejercicio

En este ejercicio vamos a publicar varias aplicaciones internas de una plataforma corporativa ficticia llamada **Aquila Digital Platform** usando **Gateway API**.

El objetivo no es solo que el acceso funcione, sino entender paso a paso la relación entre estos elementos:

```text
GatewayClass  ->  Gateway  ->  HTTPRoute  ->  Service  ->  Pod
```

Durante el laboratorio veremos:

- qué es Gateway API;
- qué papel cumple `GatewayClass`;
- qué papel cumple `Gateway`;
- cómo se usa `HTTPRoute` para enrutar tráfico HTTP;
- cómo publicar varias aplicaciones con distintos hostnames;
- cómo enrutar por path;
- cómo hacer un reparto canary 90/10 entre dos versiones de una API;
- cómo diagnosticar una ruta rota usando `kubectl describe`.

Este laboratorio está pensado para un clúster Kubernetes instalado con `kubeadm`, por ejemplo sobre instancias EC2, sin necesidad de usar `LoadBalancer` cloud ni túneles SSH.

---

## 2. Arquitectura del laboratorio

Vamos a desplegar los siguientes servicios en el namespace `aquila`:

| Hostname | Path | Servicio destino | Descripción |
|---|---|---|---|
| `aquila.local` | `/` | `portal-service` | Portal público de la plataforma |
| `aquila.local` | `/api` | `api-v1-service` y `api-v2-service` | API publicada con reparto 90/10 |
| `backoffice.aquila.local` | `/` | `backoffice-service` | Aplicación interna publicada por hostname |
| `audit.aquila.local` | `/` | `audit-service` | Servicio de auditoría usado para troubleshooting |

La arquitectura lógica será:

```text
Cliente
  |
  | HTTP Host: aquila.local / backoffice.aquila.local / audit.aquila.local
  v
NodePort 30080
  |
  v
NGINX Gateway Fabric
  |
  v
Gateway aquila-gateway
  |
  v
HTTPRoute
  |
  v
Service ClusterIP
  |
  v
Pods de aplicación
```

---

## 3. Requisitos previos

Antes de empezar, se asume que ya existe un clúster Kubernetes funcional con:

- `kubectl` configurado;
- `helm` instalado;
- conectividad a Internet desde los nodos para descargar imágenes;
- puerto `30080/TCP` permitido si se quiere probar desde fuera del clúster;
- permisos de administrador sobre el clúster.

Comprobamos la versión del clúster:

```bash
kubectl get nodes
kubectl version
```

Comprobamos Helm:

```bash
helm version
```

Limpiamos el controller del ejercicio anterior:

```bash
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace ingress-nginx
```
---

## 4. Limpieza previa opcional

Este paso solo es necesario si ya se han ejecutado pruebas anteriores de Gateway API, Envoy Gateway, NGINX Gateway Fabric o algún laboratorio previo similar.

> En un clúster de producción no se deberían borrar CRDs ni GatewayClasses sin revisar antes qué aplicaciones dependen de ellos. En este curso asumimos que estamos trabajando sobre un clúster de laboratorio.

```bash
helm uninstall eg -n envoy-gateway-system 2>/dev/null || true
helm uninstall ngf -n nginx-gateway 2>/dev/null || true
helm uninstall nginx-gateway-fabric -n nginx-gateway 2>/dev/null || true

kubectl delete namespace aquila --ignore-not-found
kubectl delete namespace kubepark --ignore-not-found
kubectl delete namespace gateway-demo --ignore-not-found
kubectl delete namespace nginx-gateway --ignore-not-found
kubectl delete namespace envoy-gateway-system --ignore-not-found

kubectl delete gatewayclass --all 2>/dev/null || true

kubectl get crd | grep -E 'gateway.networking.k8s.io|gateway.envoyproxy.io|gateway.nginx.org' | awk '{print $1}' | xargs -r kubectl delete crd

kubectl get validatingwebhookconfiguration | grep -E 'envoy|nginx|gateway' | awk '{print $1}' | xargs -r kubectl delete validatingwebhookconfiguration
kubectl get mutatingwebhookconfiguration | grep -E 'envoy|nginx|gateway' | awk '{print $1}' | xargs -r kubectl delete mutatingwebhookconfiguration
```

Comprobamos que no quedan restos relevantes:

```bash
helm list -A
kubectl get gatewayclass 2>/dev/null || true
kubectl get crd | grep -E 'gateway.networking.k8s.io|gateway.envoyproxy.io|gateway.nginx.org' || echo "No quedan CRDs de Gateway API anteriores"
```

---

## 5. Instalación de los CRDs de Gateway API

Gateway API no es simplemente un Deployment. Primero necesitamos instalar los **Custom Resource Definitions** que añaden nuevos tipos de recursos al clúster, como `GatewayClass`, `Gateway` y `HTTPRoute`.

En este laboratorio usaremos **NGINX Gateway Fabric 2.6.3**.

Definimos las versiones:

```bash
export NGF_VERSION="v2.6.3"
export NGF_HELM_VERSION="2.6.3"
```

Instalamos los CRDs estándar de Gateway API usados por NGINX Gateway Fabric:

```bash
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=${NGF_VERSION}" | kubectl apply -f -
```

Comprobamos que Kubernetes ya conoce los recursos de Gateway API:

```bash
kubectl api-resources --api-group=gateway.networking.k8s.io
```

Deberíamos ver recursos como:

```text
gatewayclasses
gateways
httproutes
tlsroutes
referencegrants
```

---

## 6. Instalación de NGINX Gateway Fabric

Ahora instalamos el controlador que será capaz de interpretar los recursos de Gateway API y configurar NGINX como plano de datos.

Usaremos `NodePort` fijo en el puerto `30080` para que el acceso sea sencillo en un laboratorio con `kubeadm`:

```bash
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --version ${NGF_HELM_VERSION} \
  --create-namespace \
  -n nginx-gateway \
  --set nginx.service.type=NodePort \
  --set-json 'nginx.service.nodePorts=[{"port":30080,"listenerPort":80}]' \
  --wait
```

Comprobamos que el controlador está desplegado:

```bash
kubectl get pods -n nginx-gateway
```

También comprobamos la `GatewayClass` creada por Helm:

```bash
kubectl get gatewayclass
```

Debe aparecer una `GatewayClass` llamada `nginx` con `ACCEPTED=True`:

```text
NAME    CONTROLLER                                   ACCEPTED
nginx   gateway.nginx.org/nginx-gateway-controller   True
```

> Importante: en este ejercicio **no vamos a crear una GatewayClass propia**. Usaremos la clase `nginx`, creada automáticamente por NGINX Gateway Fabric.

---

## 7. Despliegue de las aplicaciones Aquila

Aplicamos el namespace:

```bash
kubectl apply -f yaml/00-namespace.yaml
```

Aplicamos los Deployments y Services:

```bash
kubectl apply -f yaml/01-aplicaciones-aquila.yaml
```

Comprobamos los Pods:

```bash
kubectl get pods -n aquila
```

Esperamos a que todos estén listos:

```bash
kubectl wait --for=condition=Ready pod --all -n aquila --timeout=5m
```

Comprobamos los Services internos:

```bash
kubectl get svc -n aquila
```

Todos los Services de aplicación son `ClusterIP`. No se publican directamente hacia el exterior. El acceso externo lo gestionará Gateway API.

---

## 8. Creación del Gateway

Ahora crearemos el recurso `Gateway`.

Un `Gateway` representa el punto de entrada lógico al clúster. Define listeners, protocolos y puertos.

Aplicamos el YAML:

```bash
kubectl apply -f yaml/02-gateway.yaml
```

Consultamos el Gateway:

```bash
kubectl get gateway -n aquila
```

Debemos ver algo parecido a:

```text
NAME             CLASS   ADDRESS   PROGRAMMED
aquila-gateway   nginx             True
```

Esperamos explícitamente a que esté programado:

```bash
kubectl wait --for=condition=Programmed gateway/aquila-gateway -n aquila --timeout=5m
```

Revisamos el detalle:

```bash
kubectl describe gateway aquila-gateway -n aquila
```

Puntos importantes del YAML:

```yaml
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

La línea más importante es:

```yaml
gatewayClassName: nginx
```

Esto indica que el Gateway debe ser gestionado por la `GatewayClass` creada por NGINX Gateway Fabric.

---

## 9. Creación de la primera HTTPRoute: portal y API canary

Ahora crearemos una `HTTPRoute` para el hostname `aquila.local`.

Esta ruta hará dos cosas:

1. Las peticiones a `/` irán al portal.
2. Las peticiones a `/api` irán a dos versiones de la API, con reparto 90/10.

Aplicamos:

```bash
kubectl apply -f yaml/03-httproute-public.yaml
```

Comprobamos:

```bash
kubectl get httproute -n aquila
```

Revisamos el detalle:

```bash
kubectl describe httproute aquila-public-route -n aquila
```

El campo clave es `parentRefs`, que asocia la ruta al Gateway:

```yaml
parentRefs:
  - name: aquila-gateway
```

También usamos `hostnames`:

```yaml
hostnames:
  - aquila.local
```

Y usamos reglas por path:

```yaml
matches:
  - path:
      type: PathPrefix
      value: /api
```

La parte canary está en los pesos de los backends:

```yaml
backendRefs:
  - name: api-v1-service
    port: 80
    weight: 90
  - name: api-v2-service
    port: 80
    weight: 10
```

---

## 10. Pruebas de acceso al portal

Definimos la IP del nodo desde el que vamos a probar.

Si estás conectado por SSH al nodo master, puedes usar la IP interna del propio nodo:

```bash
export NODE_IP=$(hostname -I | awk '{print $1}')
echo $NODE_IP
```

Si vas a acceder desde tu equipo, usa la IP pública de un nodo y asegúrate de que el puerto `30080/TCP` está permitido en el Security Group o firewall.

Probamos el portal:

```bash
curl -H "Host: aquila.local" http://${NODE_IP}:30080/
```

Salida esperada:

```text
Aquila Portal | Servicio web publico | Version estable
```

---

## 11. Pruebas de acceso a la API con reparto canary

Probamos la API:

```bash
curl -H "Host: aquila.local" http://${NODE_IP}:30080/api
```

La mayoría de respuestas deberían ir a `api-v1-service`:

```text
Aquila API v1 | Servicio estable | Peso 90
```

Algunas respuestas deberían ir a `api-v2-service`:

```text
Aquila API v2 | Version canary | Peso 10
```

Para verlo mejor, lanzamos varias peticiones:

```bash
for i in {1..30}; do
  curl -s -H "Host: aquila.local" http://${NODE_IP}:30080/api
  echo
 done
```

No esperes exactamente 27 respuestas de v1 y 3 de v2. El peso define una distribución aproximada, no una secuencia exacta.

---

## 12. Creación de una segunda HTTPRoute: backoffice por hostname

Ahora publicaremos una aplicación distinta usando otro hostname.

Aplicamos:

```bash
kubectl apply -f yaml/04-httproute-backoffice.yaml
```

Comprobamos:

```bash
kubectl get httproute -n aquila
kubectl describe httproute aquila-backoffice-route -n aquila
```

Probamos:

```bash
curl -H "Host: backoffice.aquila.local" http://${NODE_IP}:30080/
```

Salida esperada:

```text
Aquila Backoffice | Aplicacion interna publicada por hostname
```

Observa que estamos usando el mismo puerto `30080`, pero el Gateway decide a qué Service enviar el tráfico en función del hostname HTTP.

---

## 13. Troubleshooting: HTTPRoute con backend incorrecto

Ahora vamos a crear una ruta con un error intencionado.

El objetivo es aprender a diagnosticar una `HTTPRoute` que existe, pero no puede resolver correctamente su backend.

Aplicamos la ruta rota:

```bash
kubectl apply -f yaml/05-httproute-audit-broken.yaml
```

Comprobamos:

```bash
kubectl get httproute -n aquila
kubectl describe httproute aquila-audit-route -n aquila
```

La ruta apunta a un Service inexistente:

```yaml
backendRefs:
  - name: audit-service-wrong
    port: 80
```

Pero el Service real se llama:

```text
audit-service
```

Probamos el acceso:

```bash
curl -i -H "Host: audit.aquila.local" http://${NODE_IP}:30080/
```

El resultado puede ser un error HTTP, normalmente relacionado con que no hay backend válido.

Ahora revisamos los Services existentes:

```bash
kubectl get svc -n aquila
```

Veremos que no existe `audit-service-wrong`, pero sí existe `audit-service`.

---

## 14. Corrección de la HTTPRoute rota

Aplicamos la ruta corregida:

```bash
kubectl apply -f yaml/06-httproute-audit-fixed.yaml
```

Comprobamos de nuevo:

```bash
kubectl describe httproute aquila-audit-route -n aquila
```

Ahora probamos:

```bash
curl -H "Host: audit.aquila.local" http://${NODE_IP}:30080/
```

Salida esperada:

```text
Aquila Audit | Servicio de auditoria | Ruta corregida
```

---

## 15. Comandos útiles de diagnóstico

Listar Gateways:

```bash
kubectl get gateway -A
```

Describir el Gateway:

```bash
kubectl describe gateway aquila-gateway -n aquila
```

Listar HTTPRoutes:

```bash
kubectl get httproute -A
```

Describir una HTTPRoute:

```bash
kubectl describe httproute aquila-public-route -n aquila
```

Ver Services de aplicación:

```bash
kubectl get svc -n aquila
```

Ver Pods:

```bash
kubectl get pods -n aquila -o wide
```

Ver la GatewayClass:

```bash
kubectl get gatewayclass
kubectl describe gatewayclass nginx
```

Buscar el Service NodePort creado para el dataplane:

```bash
kubectl get svc -A | grep 30080
```

---

## 16. Resumen conceptual

En este ejercicio hemos usado Gateway API para publicar aplicaciones HTTP en Kubernetes.

Los recursos principales han sido:

| Recurso | Función |
|---|---|
| `GatewayClass` | Define qué controlador implementa los Gateways. En este laboratorio la crea Helm y se llama `nginx`. |
| `Gateway` | Define el punto de entrada lógico, listeners, protocolos y puertos. |
| `HTTPRoute` | Define reglas HTTP: hostname, path, backend y pesos. |
| `Service` | Expone internamente los Pods dentro del clúster. |
| `Deployment` | Gestiona las réplicas de cada aplicación. |

La cadena completa ha sido:

```text
Cliente externo
  -> NodePort 30080
  -> NGINX Gateway Fabric
  -> Gateway aquila-gateway
  -> HTTPRoute
  -> Service
  -> Pod
```

Gateway API permite expresar escenarios que con Ingress clásico suelen ser más limitados o dependen de anotaciones específicas del controlador, como reparto ponderado de tráfico, separación más clara de responsabilidades y una estructura más extensible.

---

## 17. Limpieza del ejercicio

Para eliminar los recursos de la aplicación:

```bash
kubectl delete namespace aquila
```

Para eliminar NGINX Gateway Fabric:

```bash
helm uninstall ngf -n nginx-gateway
kubectl delete namespace nginx-gateway
```

Para eliminar también los CRDs de Gateway API en este clúster de laboratorio:

```bash
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.6.3" | kubectl delete -f -
```

> En un entorno compartido o productivo, no borres los CRDs de Gateway API sin comprobar antes si hay otros Gateways o Routes que dependan de ellos.
