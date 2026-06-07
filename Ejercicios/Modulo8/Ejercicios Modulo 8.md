# Ejercicios Módulo 8 - Scheduling, Jobs, HPA y afinidad

El objetivo de este módulo es practicar varios mecanismos de planificación y automatización de Kubernetes:

- `Job` y `CronJob` para tareas finitas.
- `HorizontalPodAutoscaler` para escalado automático basado en métricas.
- Etiquetas en nodos, `nodeSelector` y `nodeAffinity`.
- `podAffinity` y `podAntiAffinity` entre Pods.

> Entorno esperado: clúster Kubernetes instalado con `kubeadm`, 3 nodos EC2 `t3.medium`, Ubuntu 22.04, runtime `containerd` y red Calico.
>
> Los ejercicios evitan nombres fijos como `kubernetes-worker2`. Cada alumno elegirá los nodos reales de su clúster.

## Preparación

Crea un directorio de trabajo y copia en él los YAML de este módulo:

```bash
mkdir -p ~/modulo8
cd ~/modulo8
```

Verifica que el clúster tiene nodos `Ready`:

```bash
kubectl get nodes -o wide
```

Guarda el nombre de dos nodos worker en variables. Ajusta los nombres según la salida real de tu clúster:

```bash
kubectl get nodes

export WORKER1=<nombre-del-primer-worker>
export WORKER2=<nombre-del-segundo-worker>
```

Ejemplo:

```bash
export WORKER1=ip-10-0-1-25.eu-west-1.compute.internal
export WORKER2=ip-10-0-2-18.eu-west-1.compute.internal
```

Comprueba las variables:

```bash
echo $WORKER1
echo $WORKER2
```

## 1. Scheduling: Jobs

Un `Job` representa una tarea finita: Kubernetes crea uno o varios Pods hasta que la tarea se completa correctamente.

Aplica el manifiesto:

```bash
kubectl apply -f modulo8-job.yaml
```

Comprueba el estado:

```bash
kubectl get jobs
kubectl describe job example-job
```

Lista los Pods creados por el Job, incluyendo los completados:

```bash
kubectl get pods -l job-name=example-job
```

Obtén el nombre del Pod y consulta sus logs:

```bash
export JOB_POD=$(kubectl get pods -l job-name=example-job -o jsonpath='{.items[0].metadata.name}')
kubectl logs "$JOB_POD"
```

Deberías ver los primeros dígitos de pi.

Elimina el Job:

```bash
kubectl delete job example-job
```

Comprueba que el Pod también se elimina:

```bash
kubectl get pods -l job-name=example-job
```

> Nota: en este manifiesto se define `ttlSecondsAfterFinished: 600`. Si no eliminas el Job manualmente, Kubernetes podrá limpiarlo automáticamente después de 10 minutos una vez finalizado.

## 2. Scheduling: CronJobs

Un `CronJob` crea Jobs siguiendo una planificación en formato cron.

Aplica el manifiesto:

```bash
kubectl apply -f modulo8-cronjob.yaml
```

Comprueba el CronJob:

```bash
kubectl get cronjob hello
kubectl describe cronjob hello
```

El manifiesto usa esta planificación:

```text
*/1 * * * *
```

Esto significa: una ejecución cada minuto.

Espera hasta que se cree al menos un Job:

```bash
kubectl get jobs --watch
```

Cuando veas un Job creado por `hello`, cancela el `watch` con `Ctrl+C`.

Consulta los Pods creados por el CronJob:

```bash
kubectl get pods -l job-name
```

Obtén el último Pod creado y revisa sus logs:

```bash
export CRON_POD=$(kubectl get pods --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1:].metadata.name}')
kubectl logs "$CRON_POD"
```

Limpia el CronJob y los Jobs creados:

```bash
kubectl delete cronjob hello
kubectl delete jobs -l job-name --ignore-not-found=true
```

Si quedan Pods completados, puedes eliminarlos con:

```bash
kubectl delete pods --field-selector=status.phase=Succeeded
```

## 3. Escalado automático con HPA

Para que `HorizontalPodAutoscaler` funcione, el clúster necesita `metrics-server` operativo.

### 3.1 Instalar o verificar metrics-server

Comprueba si ya existe:

```bash
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
```

Si `kubectl top nodes` devuelve métricas, puedes continuar.

Si no está instalado, aplica el manifiesto incluido:

```bash
kubectl apply -f modulo8-metrics-server-components.yaml
```

Espera a que esté disponible:

```bash
kubectl rollout status deployment/metrics-server -n kube-system
kubectl get apiservice v1beta1.metrics.k8s.io
```

Prueba de nuevo:

```bash
kubectl top nodes
kubectl top pods -A
```

> En muchos clústeres `kubeadm` de laboratorio se necesita el argumento `--kubelet-insecure-tls` porque el certificado del kubelet no siempre incluye los SAN esperados. Por eso está incluido en el YAML.

### 3.2 Desplegar la aplicación web

Aplica el despliegue y el Service:

```bash
kubectl apply -f modulo8-web.yaml
```

Comprueba el estado:

```bash
kubectl get deployment web
kubectl get service web
kubectl get pods -l app=web -o wide
```

El Service es de tipo `NodePort` y usa el puerto `30081`. Desde fuera del clúster, asegúrate de que el Security Group permite TCP/30081 si quieres probar con la IP pública de un nodo:

```bash
curl http://<IP_PUBLICA_DE_UN_NODO>:30081
```

Desde dentro del clúster puedes probar con:

```bash
kubectl run tmp-curl --rm -it --image=curlimages/curl:8.8.0 --restart=Never -- curl http://web
```

### 3.3 Crear el HPA

Aplica el HPA:

```bash
kubectl apply -f modulo8-hpa.yaml
```

Comprueba su estado:

```bash
kubectl get hpa
kubectl describe hpa web
```

Al principio puede aparecer `TARGETS: <unknown>/20%`. Espera a que `metrics-server` recoja métricas:

```bash
watch kubectl get hpa
```

Cancela con `Ctrl+C`.

### 3.4 Generar carga

Aplica el generador de carga:

```bash
kubectl apply -f modulo8-loadgen.yaml
```

Observa el HPA y los Pods:

```bash
watch 'kubectl get hpa; echo; kubectl get deployment web; echo; kubectl get pods -l app=web -o wide'
```

El HPA debería aumentar gradualmente las réplicas de `web` hasta un máximo de 4, si hay CPU suficiente y las métricas se recogen correctamente.

Detén la carga:

```bash
kubectl scale deployment loadgen --replicas=0
```

La reducción de réplicas puede tardar. En este manifiesto se ha configurado una ventana de estabilización de 60 segundos para que el laboratorio sea más ágil:

```bash
kubectl describe hpa web
```

Limpieza de esta parte:

```bash
kubectl delete -f modulo8-loadgen.yaml --ignore-not-found=true
kubectl delete -f modulo8-hpa.yaml --ignore-not-found=true
kubectl delete -f modulo8-web.yaml --ignore-not-found=true
kubectl delete pod tmp-curl --ignore-not-found=true
```

## 4. Agrupar nodos con etiquetas y nodeSelector

Primero, elige un worker real:

```bash
echo $WORKER2
```

Añade una etiqueta de ejemplo:

```bash
kubectl label node "$WORKER2" workload=production --overwrite
```

Filtra nodos por esa etiqueta:

```bash
kubectl get nodes -l workload=production
```

Cambia el valor:

```bash
kubectl label node "$WORKER2" workload=dev --overwrite
kubectl get nodes -l workload=dev
```

Quita la etiqueta:

```bash
kubectl label node "$WORKER2" workload-
```

Ahora añade la etiqueta que usará el `nodeSelector`:

```bash
kubectl get nodes --selector disktype=ssd
kubectl label node "$WORKER2" disktype=ssd --overwrite
kubectl get nodes --selector disktype=ssd
```

Aplica el Pod con `nodeSelector`:

```bash
kubectl apply -f modulo8-pod-nodeselector.yaml
```

Comprueba en qué nodo se ha planificado:

```bash
kubectl get pod pod-nodeselector -o wide
```

Debe estar en `$WORKER2`.

Si el Pod queda en `Pending`, revisa los eventos:

```bash
kubectl describe pod pod-nodeselector
```

Limpieza:

```bash
kubectl delete -f modulo8-pod-nodeselector.yaml
```

## 5. Afinidad de nodo

`nodeSelector` solo permite coincidencias simples. `nodeAffinity` permite reglas más expresivas con operadores como `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt` y `Lt`.

Etiqueta un nodo con una zona ficticia:

```bash
kubectl label node "$WORKER2" azname=az1 --overwrite
```

Aplica el Pod con afinidad:

```bash
kubectl apply -f modulo8-pod-node-affinity.yaml
kubectl get pod pod-node-affinity -o wide
```

El Pod solo puede programarse en nodos con `azname=az1` o `azname=az2`. Además, si hay varios candidatos, preferirá nodos con `disktype=ssd`.

Borra el Pod y mueve la etiqueta a otro nodo:

```bash
kubectl delete -f modulo8-pod-node-affinity.yaml
kubectl label node "$WORKER2" azname-
kubectl label node "$WORKER1" azname=az1 --overwrite
```

Aplica de nuevo:

```bash
kubectl apply -f modulo8-pod-node-affinity.yaml
kubectl get pod pod-node-affinity -o wide
```

Comprueba que ahora se planifica en `$WORKER1`.

Limpieza:

```bash
kubectl delete -f modulo8-pod-node-affinity.yaml
kubectl label node "$WORKER1" azname-
kubectl label node "$WORKER2" disktype-
```

## 6. Antiafinidad entre Pods

En este ejercicio opcional se despliegan dos réplicas de Redis y dos réplicas de nginx.

El Deployment `redis-store` usa `podAntiAffinity` para evitar que dos Pods `app=store` caigan en el mismo nodo.

El Deployment `web-affinity` usa:

- `podAffinity` para intentar colocarse en nodos donde exista un Pod `app=store`.
- `podAntiAffinity` para evitar que dos Pods `app=web-affinity` caigan en el mismo nodo.

Aplica Redis:

```bash
kubectl apply -f modulo8-redis-affinity.yaml
kubectl rollout status deployment/redis-store
kubectl get pods -l app=store -o wide
```

Aplica la aplicación web:

```bash
kubectl apply -f modulo8-web-affinity.yaml
kubectl rollout status deployment/web-affinity
kubectl get pods -l 'app in (store,web-affinity)' -o wide
```

En un clúster con al menos dos workers disponibles deberías ver una distribución parecida a:

```text
redis-store-...     NODE worker-a
web-affinity-...    NODE worker-a
redis-store-...     NODE worker-b
web-affinity-...    NODE worker-b
```

Es decir, cada web queda junto a un Redis, pero no se colocan dos Redis ni dos web iguales en el mismo nodo.

Si no hay nodos suficientes, algunos Pods quedarán en `Pending`. Compruébalo con:

```bash
kubectl describe pod <pod-pending>
```

Limpieza:

```bash
kubectl delete -f modulo8-web-affinity.yaml --ignore-not-found=true
kubectl delete -f modulo8-redis-affinity.yaml --ignore-not-found=true
```

## Limpieza final del módulo

Ejecuta estos comandos para dejar el clúster limpio:

```bash
kubectl delete job example-job --ignore-not-found=true
kubectl delete cronjob hello --ignore-not-found=true
kubectl delete jobs --all --ignore-not-found=true
kubectl delete pods --field-selector=status.phase=Succeeded --ignore-not-found=true
kubectl delete -f modulo8-loadgen.yaml --ignore-not-found=true
kubectl delete -f modulo8-hpa.yaml --ignore-not-found=true
kubectl delete -f modulo8-web.yaml --ignore-not-found=true
kubectl delete -f modulo8-pod-nodeselector.yaml --ignore-not-found=true
kubectl delete -f modulo8-pod-node-affinity.yaml --ignore-not-found=true
kubectl delete -f modulo8-web-affinity.yaml --ignore-not-found=true
kubectl delete -f modulo8-redis-affinity.yaml --ignore-not-found=true
kubectl label node "$WORKER1" azname- --overwrite 2>/dev/null || true
kubectl label node "$WORKER2" azname- disktype- workload- --overwrite 2>/dev/null || true
```

No elimines `metrics-server` si vas a usarlo en módulos posteriores:

```bash
# Solo si quieres desinstalarlo:
# kubectl delete -f modulo8-metrics-server-components.yaml
```

## Resumen de correcciones respecto al ejercicio original

- Se eliminan dependencias de manifiestos remotos en S3.
- Se evita usar nombres de nodo fijos como `kubernetes-worker1` o `kubernetes-worker2`.
- Se añade una instalación local de `metrics-server` adaptada a clústeres `kubeadm` de laboratorio.
- Se corrige el ejemplo de HPA para usar `autoscaling/v2`, métrica de CPU de tipo `Resource` y `behavior` válido.
- Se sustituye el puerto 8080 por el puerto real de la imagen `registry.k8s.io/hpa-example`, que escucha en HTTP/80.
- Se añade limpieza de recursos para evitar interferencias con módulos posteriores.
