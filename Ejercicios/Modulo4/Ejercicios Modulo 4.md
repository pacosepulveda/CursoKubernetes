# Ejercicios Módulo 4

## Objetivo

En este módulo vamos a practicar tareas básicas de operación de Kubernetes:

- Consulta de logs de contenedores.
- Uso de Deployments y ReplicaSets.
- Recuperación automática de Pods gestionados por un Deployment.
- Uso de Namespaces.
- Uso de Labels y Selectors.

Los ejercicios están pensados para un cluster Kubernetes instalado con `kubeadm`, usando `containerd` y Calico como CNI.

> Nota: las salidas esperadas son orientativas. Los nombres de Pods, hashes de ReplicaSets, IPs, nodos y tiempos cambiaran en cada entorno.

## Archivos necesarios

Copia en el nodo desde el que ejecutes `kubectl` estos archivos:

- `modulo4-logging.yaml`
- `modulo4-nginx-deployment.yaml`
- `modulo4-labels.yaml`

Comprueba primero que tienes acceso al cluster:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

---

# 1. Logging

El logging básico en Kubernetes permite consultar la salida `stdout` y `stderr` de los contenedores mediante `kubectl logs`.

En este ejercicio crearemos tres Pods:

- `logme`: genera logs continuamente por `stdout` y `stderr`.
- `oneshot`: escribe unas líneas y termina correctamente.
- `restart-logs`: termina con error y se reinicia, para poder consultar logs de la instancia anterior del contenedor con `--previous`.

Crea los Pods:

```bash
kubectl apply -f modulo4-logging.yaml
```

Comprueba su estado:

```bash
kubectl get pods
```

Salida esperada aproximada:

```text
NAME           READY   STATUS             RESTARTS      AGE
logme          1/1     Running            0             20s
oneshot        0/1     Completed          0             20s
restart-logs   0/1     CrashLoopBackOff   1             20s
```

El estado de `restart-logs` puede alternar entre `Error`, `CrashLoopBackOff` y `Running` mientras Kubernetes lo reinicia.

## 1.1 Consultar las últimas líneas de log

Muestra las últimas 5 líneas del contenedor `gen` del Pod `logme`:

```bash
kubectl logs --tail=5 logme -c gen
```

Salida esperada aproximada:

```text
stdout log line 12
stderr log line 12
stdout log line 13
stderr log line 13
stdout log line 14
```

## 1.2 Seguir logs en tiempo real

Ejecuta:

```bash
kubectl logs -f --since=10s logme -c gen
```

Deberías ver nuevas líneas cada pocos segundos.

Pulsa `Ctrl+C` para salir.

## 1.3 Consultar logs de un Pod completado

El Pod `oneshot` ya ha terminado, pero sus logs siguen disponibles mientras el Pod exista:

```bash
kubectl logs oneshot -c gen
```

Salida esperada aproximada:

```text
countdown: 9
countdown: 8
countdown: 7
...
countdown: 1
finished
```

> Importante: no uses `-p` o `--previous` para este caso. `--previous` sirve para consultar la instancia anterior de un contenedor que se ha reiniciado, no para consultar simplemente un Pod completado.

## 1.4 Consultar logs de la instancia anterior de un contenedor

El Pod `restart-logs` falla y Kubernetes lo reinicia. Espera a que tenga al menos un reinicio:

```bash
kubectl get pod restart-logs -w
```

Cuando veas que `RESTARTS` es mayor que 0, pulsa `Ctrl+C` y consulta los logs de la instancia anterior:

```bash
kubectl logs --previous restart-logs -c gen
```

También puedes usar la forma corta:

```bash
kubectl logs -p restart-logs -c gen
```

Salida esperada aproximada:

```text
starting container
simulating an application error
```

## 1.5 Limpiar los Pods de logging

```bash
kubectl delete -f modulo4-logging.yaml
```

---

# 2. Deployments

Un Pod es la unidad mínima de ejecución en Kubernetes, pero normalmente no se crean Pods sueltos para aplicaciones reales. Lo habitual es crear objetos superiores como Deployments.

Un Deployment gestiona ReplicaSets, y un ReplicaSet mantiene el número deseado de Pods.

## 2.1 Crear un Deployment de forma imperativa

Crea un Deployment llamado `multitool`:

```bash
kubectl create deployment multitool --image=ghcr.io/eficode-academy/network-multitool
```

Comprueba los objetos creados:

```bash
kubectl get deployment,replicaset,pod -l app=multitool
```

Salida esperada aproximada:

```text
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/multitool   1/1     1            1           30s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/multitool-xxxxxxxxxx   1         1         1       30s

NAME                             READY   STATUS    RESTARTS   AGE
pod/multitool-xxxxxxxxxx-yyyyy   1/1     Running   0          30s
```

> Nota: en versiones modernas de Kubernetes verás recursos como `deployment.apps` y `replicaset.apps`, no `deployment.extensions` ni `replicaset.extensions`.

## 2.2 Crear un Deployment desde YAML

Primero crea un Deployment de nginx de forma imperativa para ver la diferencia:

```bash
kubectl create deployment nginx --image=nginx:1.27-alpine
```

Comprueba los objetos:

```bash
kubectl get pods,deployments,replicasets -l app=nginx
```

Ahora elimina el Deployment para recrearlo desde YAML:

```bash
kubectl delete deployment nginx
```

Crea el Deployment desde el archivo local:

```bash
kubectl apply -f modulo4-nginx-deployment.yaml
```

Comprueba que se ha creado:

```bash
kubectl get deployments
kubectl get pods -l app=nginx -o wide
```

Salida esperada aproximada:

```text
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
multitool   1/1     1            1           5m
nginx       1/1     1            1           30s
```

## 2.3 Comprobar la reconciliación del Deployment

Un error habitual es borrar manualmente un Pod gestionado por un Deployment esperando que la aplicación desaparezca. Kubernetes volverá a crear otro Pod para cumplir el estado deseado.

Guarda el nombre del Pod de nginx:

```bash
NGINX_POD=$(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}')
echo "$NGINX_POD"
```

Elimina ese Pod:

```bash
kubectl delete pod "$NGINX_POD"
```

Observa cómo se crea otro Pod:

```bash
kubectl get pods -l app=nginx -w
```

Cuando veas un nuevo Pod en estado `Running`, pulsa `Ctrl+C`.

La explicación es que no hemos eliminado el Deployment ni el ReplicaSet, solo una réplica concreta.

Compruebalo:

```bash
kubectl get deployment,replicaset,pod -l app=nginx
```

---

# 3. Namespaces

Los Namespaces permiten separar recursos dentro del mismo cluster. Son útiles para organizar entornos, equipos o ejercicios.

Crea un Namespace:

```bash
kubectl create namespace my-namespace
```

Comprueba que existe:

```bash
kubectl get namespaces
```

Ejecuta una consulta contra el Namespace `default`:

```bash
kubectl get pods -n default
```

Ejecuta una consulta contra el nuevo Namespace:

```bash
kubectl get pods -n my-namespace
```

Salida esperada:

```text
No resources found in my-namespace namespace.
```

## 3.1 Cambiar el Namespace por defecto del contexto actual

Puedes configurar `kubectl` para que use `my-namespace` por defecto en el contexto actual:

```bash
kubectl config set-context --current --namespace=my-namespace
```

Comprueba el contexto actual:

```bash
kubectl config get-contexts
```

Ahora este comando consulta `my-namespace`:

```bash
kubectl get pods
```

Para no afectar a los siguientes ejercicios, vuelve a dejar `default` como Namespace por defecto:

```bash
kubectl config set-context --current --namespace=default
```

Comprueba de nuevo:

```bash
kubectl config get-contexts
```

---

# 4. Labels

Las Labels son pares clave-valor que permiten organizar y seleccionar objetos de Kubernetes.

Crea dos Pods con distintas etiquetas:

```bash
kubectl apply -f modulo4-labels.yaml
```

Comprueba las etiquetas:

```bash
kubectl get pods --show-labels
```

Salida esperada aproximada:

```text
NAME           READY   STATUS    RESTARTS   AGE   LABELS
labelex        1/1     Running   0          10s   env=development
labelexother   1/1     Running   0          10s   env=production,owner=michael
```

## 4.1 Agregar una Label

Agrega la etiqueta `owner=user1` al Pod `labelex`:

```bash
kubectl label pod labelex owner=user1
```

Comprueba el resultado:

```bash
kubectl get pods --show-labels
```

Salida esperada aproximada:

```text
NAME           READY   STATUS    RESTARTS   AGE   LABELS
labelex        1/1     Running   0          1m    env=development,owner=user1
labelexother   1/1     Running   0          1m    env=production,owner=michael
```

## 4.2 Filtrar por Label

Lista solo los Pods cuyo propietario sea `user1`:

```bash
kubectl get pods --selector owner=user1
```

Forma abreviada equivalente:

```bash
kubectl get pods -l owner=user1
```

Lista los Pods con `env=development`:

```bash
kubectl get pods -l env=development
```

## 4.3 Selectores basados en conjuntos

Lista los Pods cuyo `env` sea `development` o `production`:

```bash
kubectl get pods -l 'env in (development,production)'
```

Salida esperada aproximada:

```text
NAME           READY   STATUS    RESTARTS   AGE
labelex        1/1     Running   0          3m
labelexother   1/1     Running   0          3m
```

## 4.4 Eliminar recursos usando Labels

Elimina los Pods creados en este apartado usando un selector:

```bash
kubectl delete pods -l 'env in (development,production)'
```

Comprueba que ya no existen:

```bash
kubectl get pods -l 'env in (development,production)'
```

---

# 5. Limpieza final

Elimina los Deployments creados:

```bash
kubectl delete deployment nginx multitool --ignore-not-found
```

Elimina el Namespace del ejercicio:

```bash
kubectl delete namespace my-namespace --ignore-not-found
```

Comprueba el estado final:

```bash
kubectl get pods
kubectl get deployments
```

