# Ejercicios Módulo 5

## Despliegues, actualizaciones, Service, Canary, StatefulSet y DaemonSet

En este módulo practicaremos con varios controladores y objetos habituales de Kubernetes:

- `Deployment`
- `ReplicaSet`
- `Service`
- despliegues canary sencillos
- `StatefulSet`
- `DaemonSet`

> Entorno esperado: clúster Kubernetes instalado con `kubeadm`, runtime `containerd`, CNI Calico y 3 nodos EC2 `t3.medium` con Ubuntu 22.04.

---

## 0. Preparación

Archivos necesarios:

```text
modulo5-nginx-deployment.yaml
modulo5-nginx-service.yaml
modulo5-nginx-canary.yaml
modulo5-statefulset.yaml
modulo5-daemonset-error.yaml
modulo5-daemonset-running.yaml
```

Comprueba que el clúster está operativo:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

---

## 1. Deployment inicial de nginx

Un `Deployment` gestiona la creación y actualización de Pods mediante `ReplicaSet`.

Aplica el manifiesto inicial:

```bash
kubectl apply -f modulo5-nginx-deployment.yaml
```

Comprueba el Deployment:

```bash
kubectl get deployments
```

Salida esperada aproximada:

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           30s
```

Comprueba los Pods y los nodos donde se han planificado:

```bash
kubectl get pods -l app=nginx -o wide
```

Comprueba los objetos relacionados:

```bash
kubectl get deployment,replicaset,pod -l app=nginx
```

> Nota: los nombres de los Pods y ReplicaSets serán diferentes en cada ejecución.

---

## 2. Escalado de un Deployment

Reduce el Deployment a una réplica:

```bash
kubectl scale deployment/nginx-deployment --replicas=1
kubectl get pods -l app=nginx -o wide
```

Devuélvelo a tres réplicas:

```bash
kubectl scale deployment/nginx-deployment --replicas=3
kubectl get pods -l app=nginx -o wide
```

Observa que escalar no crea una nueva revisión de rollout, porque no cambia la plantilla del Pod (`.spec.template`).

```bash
kubectl rollout history deployment/nginx-deployment
```

---

## 3. Actualización de imagen y rollout

Primero anotamos la causa del cambio. El flag `--record` está obsoleto, así que usaremos la anotación estándar `kubernetes.io/change-cause`.

```bash
kubectl annotate deployment/nginx-deployment \
  kubernetes.io/change-cause="Imagen inicial nginx:1.25-alpine" \
  --overwrite
```

Actualiza la imagen del contenedor `nginx`:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.27-alpine
```

Anota la causa del cambio:

```bash
kubectl annotate deployment/nginx-deployment \
  kubernetes.io/change-cause="Actualización a nginx:1.27-alpine" \
  --overwrite
```

Consulta el estado del rollout:

```bash
kubectl rollout status deployment/nginx-deployment
```

Comprueba que los Pods nuevos usan la imagen actualizada:

```bash
kubectl get pods -l app=nginx -o wide
kubectl describe deployment nginx-deployment | grep -i image
```

Consulta el historial:

```bash
kubectl rollout history deployment/nginx-deployment
```

Para ver una revisión concreta:

```bash
kubectl rollout history deployment/nginx-deployment --revision=1
kubectl rollout history deployment/nginx-deployment --revision=2
```

---

## 4. Rollback

Revierte el Deployment a la revisión anterior:

```bash
kubectl rollout undo deployment/nginx-deployment
kubectl rollout status deployment/nginx-deployment
```

Comprueba el historial:

```bash
kubectl rollout history deployment/nginx-deployment
```

Comprueba de nuevo la imagen:

```bash
kubectl describe deployment nginx-deployment | grep -i image
```

Si hubiera varias revisiones, podrías volver a una concreta con:

```bash
kubectl rollout undo deployment/nginx-deployment --to-revision=REVISION_NUMBER
```

---

## 5. Service para exponer nginx

En un clúster kubeadm sobre EC2, un `Service` de tipo `LoadBalancer` normalmente no crea automáticamente un balanceador externo de AWS salvo que exista un cloud-controller-manager correctamente instalado y configurado.

Para que el ejercicio sea reproducible, usaremos un `NodePort`.

Aplica el Service:

```bash
kubectl apply -f modulo5-nginx-service.yaml
```

Comprueba el Service:

```bash
kubectl get service nginx
```

Salida esperada aproximada:

```text
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.96.x.y       <none>        80:30080/TCP   10s
```

Prueba el acceso desde dentro del clúster:

```bash
kubectl run curl --rm -it --restart=Never --image=curlimages/curl:8.8.0 -- \
  curl -s http://nginx.default.svc.cluster.local
```

También puedes probar usando la IP interna de cualquier nodo:

```bash
kubectl get nodes -o wide
curl http://IP_INTERNA_DEL_NODO:30080
```

> En AWS, para acceder desde tu equipo local al puerto `30080`, el Security Group de los nodos debe permitir entrada TCP a ese puerto desde tu IP.

---

## 6. Canary simple con un único Service

Un despliegue canary puede consistir en tener dos Deployments diferentes detrás del mismo Service.

En este ejercicio:

- `nginx-deployment` representa la versión estable.
- `nginx-canary` representa la versión canary.
- El Service `nginx` selecciona ambos porque ambos tienen la etiqueta `app: nginx`.

Aplica el Deployment canary:

```bash
kubectl apply -f modulo5-nginx-canary.yaml
```

Comprueba los Deployments:

```bash
kubectl get deployments
```

Comprueba todos los Pods seleccionados por el Service:

```bash
kubectl get pods -l app=nginx -o wide --show-labels
```

Comprueba los endpoints del Service:

```bash
kubectl get endpoints nginx
```

O, en versiones recientes de Kubernetes:

```bash
kubectl get endpointslice -l kubernetes.io/service-name=nginx
```

Haz varias peticiones al Service desde dentro del clúster:

```bash
kubectl run curl --rm -it --restart=Never --image=curlimages/curl:8.8.0 -- sh
```

Dentro del contenedor:

```sh
for i in $(seq 1 10); do curl -s -o /dev/null -w "%{http_code}\n" http://nginx.default.svc.cluster.local; done
exit
```

Ahora reduce a cero la versión estable:

```bash
kubectl scale deployment/nginx-deployment --replicas=0
kubectl get deployments
kubectl get pods -l app=nginx -o wide --show-labels
```

El Service seguirá funcionando porque todavía hay Pods canary con la etiqueta `app=nginx`.

Prueba de nuevo:

```bash
kubectl run curl --rm -it --restart=Never --image=curlimages/curl:8.8.0 -- \
  curl -I http://nginx.default.svc.cluster.local
```

Vuelve a escalar la versión estable para dejar el entorno como estaba:

```bash
kubectl scale deployment/nginx-deployment --replicas=3
```

---

## 7. StatefulSet sin almacenamiento persistente

Un `StatefulSet` gestiona Pods con identidad estable y nombres ordenados. En este ejercicio lo desplegaremos sin `PersistentVolumeClaim`, porque el almacenamiento persistente se verá en otro módulo.

Aplica el manifiesto:

```bash
kubectl apply -f modulo5-statefulset.yaml
```

Comprueba el StatefulSet:

```bash
kubectl get statefulset
```

Observa los Pods:

```bash
kubectl get pods -l app=web-stateful -o wide
```

Deberías ver nombres ordenados:

```text
web-0
web-1
web-2
```

Comprueba que no se ha creado ningún ReplicaSet para este workload:

```bash
kubectl get replicasets
```

Elimina uno de los Pods:

```bash
kubectl delete pod web-1
```

Comprueba que Kubernetes vuelve a crear un Pod con el mismo nombre:

```bash
kubectl get pods -l app=web-stateful -o wide
```

> Aunque el nombre se mantiene, al no haber almacenamiento persistente los datos escritos dentro del contenedor se perderán si el Pod se recrea.

---

## 8. DaemonSet que termina inmediatamente

Un `DaemonSet` asegura que haya un Pod en cada nodo elegible. Es habitual para agentes de logging, monitorización, red o almacenamiento.

El siguiente DaemonSet ejecuta un comando que termina inmediatamente:

```bash
kubectl apply -f modulo5-daemonset-error.yaml
```

Comprueba los Pods:

```bash
kubectl get pods -l app=hello-daemonset-error -o wide
```

Es probable que veas estados como `Completed`, `CrashLoopBackOff` o múltiples reinicios, dependiendo del momento en que ejecutes el comando.

Consulta los logs:

```bash
kubectl logs -l app=hello-daemonset-error --tail=20
```

Consulta la descripción de un Pod:

```bash
POD=$(kubectl get pod -l app=hello-daemonset-error -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod "$POD"
```

### Pregunta

¿Están en ejecución los Pods? ¿Cuál es el motivo?

### Respuesta esperada

No son Pods útiles para un DaemonSet de larga duración porque el comando del contenedor hace `echo` y termina. Kubernetes intenta mantener el Pod definido por el DaemonSet, pero el proceso principal del contenedor finaliza. Para que el Pod permanezca en `Running`, el contenedor debe ejecutar un proceso persistente.

Elimina este DaemonSet antes de continuar:

```bash
kubectl delete -f modulo5-daemonset-error.yaml
```

---

## 9. DaemonSet que permanece en ejecución

Aplica ahora una versión que mantiene un proceso en primer plano:

```bash
kubectl apply -f modulo5-daemonset-running.yaml
```

Comprueba que hay un Pod por nodo elegible:

```bash
kubectl get daemonset
kubectl get pods -l app=hello-daemonset-running -o wide
```

Consulta los logs:

```bash
kubectl logs -l app=hello-daemonset-running --tail=20
```

En un clúster de 3 nodos, si el control-plane no tiene taints que impidan planificar Pods normales, verás 3 Pods. Si el nodo control-plane conserva el taint habitual `node-role.kubernetes.io/control-plane:NoSchedule`, el DaemonSet se ejecutará solo en los workers, salvo que se añadan tolerations.

Comprueba los taints:

```bash
kubectl describe nodes | grep -i taint -A1
```

---

## 10. Limpieza

Cuando termines el módulo, elimina los recursos creados:

```bash
kubectl delete -f modulo5-daemonset-running.yaml --ignore-not-found
kubectl delete -f modulo5-daemonset-error.yaml --ignore-not-found
kubectl delete -f modulo5-statefulset.yaml --ignore-not-found
kubectl delete -f modulo5-nginx-canary.yaml --ignore-not-found
kubectl delete -f modulo5-nginx-service.yaml --ignore-not-found
kubectl delete -f modulo5-nginx-deployment.yaml --ignore-not-found
kubectl delete pod curl --ignore-not-found
```

Comprueba que no quedan recursos principales del módulo:

```bash
kubectl get deployment,statefulset,daemonset,service,pod
```
