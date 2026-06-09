# Solución - Ejercicio 02 - StatefulSet y DaemonSet

## Parte 1 - Crear el StatefulSet

Aplicamos el manifiesto:

```bash
kubectl apply -f yaml/modulo5-ejercicio02-statefulset.yaml
```

Salida esperada:

```text
statefulset.apps/contador created
```

Comprobamos el StatefulSet:

```bash
kubectl get statefulset contador
```

Salida esperada aproximada:

```text
NAME       READY   AGE
contador   3/3     1m
```

Comprobamos los Pods:

```bash
kubectl get pods -l app=contador -o wide
```

Salida esperada aproximada:

```text
NAME         READY   STATUS    RESTARTS   AGE   IP            NODE
contador-0   1/1     Running   0          1m    192.168.x.x   nodo-1
contador-1   1/1     Running   0          1m    192.168.x.x   nodo-2
contador-2   1/1     Running   0          1m    192.168.x.x   nodo-3
```

Consultamos logs:

```bash
kubectl logs contador-0 --tail=5
kubectl logs contador-1 --tail=5
kubectl logs contador-2 --tail=5
```

Cada Pod mostrará mensajes con su propio nombre de host.

### Respuestas

1. Los Pods se llaman `contador-0`, `contador-1` y `contador-2`.
2. Los nombres no son aleatorios. Siguen un patrón estable y ordenado.
3. No se crea un ReplicaSet para este StatefulSet.

Lo comprobamos con:

```bash
kubectl get replicasets
```

Un `StatefulSet` gestiona directamente sus Pods y les asigna una identidad estable. En cambio, un `Deployment` utiliza ReplicaSets.

## Parte 2 - Eliminar un Pod del StatefulSet

Eliminamos un Pod:

```bash
kubectl delete pod contador-1
```

Comprobamos de nuevo:

```bash
kubectl get pods -l app=contador -o wide
```

Kubernetes volverá a crear un Pod llamado `contador-1`.

Consultamos logs:

```bash
kubectl logs contador-1 --tail=5
```

### Respuestas

1. Sí, Kubernetes vuelve a crear el Pod porque el StatefulSet mantiene el número deseado de réplicas.
2. Sí, el nuevo Pod conserva el mismo nombre: `contador-1`.
3. El contador empieza de nuevo.
4. Se pierde el estado interno porque en este ejercicio no hay almacenamiento persistente. El contador vive solo en el proceso del contenedor. Al eliminarse el Pod, se elimina el contenedor y se pierde ese estado.

Este punto es importante: un `StatefulSet` da identidad estable, pero el almacenamiento persistente requiere volúmenes persistentes, que se trabajan en otros módulos.

## Parte 3 - Escalar el StatefulSet

Escalamos a 4 réplicas:

```bash
kubectl scale statefulset/contador --replicas=4
```

Comprobamos:

```bash
kubectl get pods -l app=contador -o wide
```

Debería aparecer:

```text
contador-0
contador-1
contador-2
contador-3
```

Reducimos a 2 réplicas:

```bash
kubectl scale statefulset/contador --replicas=2
```

Comprobamos:

```bash
kubectl get pods -l app=contador -o wide
```

Deberían quedar:

```text
contador-0
contador-1
```

### Respuestas

1. Al escalar de 3 a 4 se crea `contador-3`.
2. Al reducir de 4 a 2 desaparecen primero los Pods con ordinal más alto: `contador-3` y `contador-2`.
3. La diferencia principal frente a un Deployment es que el StatefulSet mantiene nombres ordenados y una identidad estable por réplica. En un Deployment, los nombres de los Pods incluyen sufijos generados y no representan una identidad estable de aplicación.

## Parte 4 - Crear un DaemonSet

Aplicamos el DaemonSet:

```bash
kubectl apply -f yaml/modulo5-ejercicio02-daemonset.yaml
```

Comprobamos:

```bash
kubectl get daemonset auditor-nodo
```

Salida esperada aproximada:

```text
NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
auditor-nodo   2         2         2       2            2           <none>          1m
```

El número exacto dependerá de los nodos elegibles.

Comprobamos los Pods:

```bash
kubectl get pods -l app=auditor-nodo -o wide
```

Consultamos logs:

```bash
kubectl logs -l app=auditor-nodo --tail=10
```

Comprobamos nodos:

```bash
kubectl get nodes
```

Comprobamos taints:

```bash
kubectl describe nodes | grep -i taint -A1
```

### Respuestas

1. El DaemonSet crea un Pod por cada nodo elegible.
2. Puede coincidir o no con el número total de nodos.
3. Si no coincide, normalmente el nodo excluido será el `control-plane`, porque suele tener un taint `NoSchedule` que impide planificar Pods normales.

Un DaemonSet no ignora automáticamente los taints. Para ejecutarse en nodos con taints, el Pod del DaemonSet debe incluir `tolerations`.

## Parte 5 - DaemonSet con toleration para el control-plane

Aplicamos el segundo DaemonSet:

```bash
kubectl apply -f yaml/modulo5-ejercicio02-daemonset-control-plane.yaml
```

Comprobamos los Pods:

```bash
kubectl get pods -l app=auditor-nodo-control-plane -o wide
```

Si el nodo de control-plane tenía el taint habitual, ahora debería aparecer también un Pod en ese nodo.

### Respuestas

1. Sí, si el problema era el taint del control-plane, el nuevo DaemonSet podrá crear un Pod también en ese nodo.
2. La sección `tolerations` permite que el Pod tolere determinados taints del nodo.
3. Es necesario en DaemonSets de monitorización, logging o red cuando queremos ejecutar un agente también en nodos del plano de control.

Ejemplos típicos:

- Agentes de recogida de logs.
- Agentes de métricas.
- Componentes de red.
- Herramientas de observabilidad por nodo.

## Limpieza

Eliminamos los recursos:

```bash
kubectl delete -f yaml/modulo5-ejercicio02-daemonset-control-plane.yaml --ignore-not-found
kubectl delete -f yaml/modulo5-ejercicio02-daemonset.yaml --ignore-not-found
kubectl delete -f yaml/modulo5-ejercicio02-statefulset.yaml --ignore-not-found
```

Comprobamos:

```bash
kubectl get pods -l app=contador
kubectl get pods -l app=auditor-nodo
kubectl get pods -l app=auditor-nodo-control-plane
```

No deberían quedar Pods asociados a estos ejercicios.
