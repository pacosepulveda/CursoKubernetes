# Ejercicio 02 - StatefulSet y DaemonSet

## Objetivo

Practicar dos controladores de Kubernetes con comportamientos diferentes:

- `StatefulSet`: Pods con identidad estable y nombres ordenados.
- `DaemonSet`: ejecución de un Pod en cada nodo elegible del clúster.

## Archivos proporcionados

En la carpeta `yaml/` tienes estos archivos:

```text
modulo5-ejercicio02-statefulset.yaml
modulo5-ejercicio02-daemonset.yaml
modulo5-ejercicio02-daemonset-control-plane.yaml
```

## Contexto

El `StatefulSet` `contador` ejecuta tres Pods que escriben un mensaje periódico en los logs. Cada Pod tendrá un nombre estable:

```text
contador-0
contador-1
contador-2
```

El `DaemonSet` `auditor-nodo` ejecuta un Pod en cada nodo elegible. En un clúster de 3 nodos, el número de Pods dependerá de si el nodo de control-plane permite o no planificar Pods normales.

## Parte 1 - Crear el StatefulSet

Aplica el StatefulSet:

```bash
kubectl apply -f yaml/modulo5-ejercicio02-statefulset.yaml
```

Comprueba el StatefulSet:

```bash
kubectl get statefulset contador
```

Comprueba los Pods:

```bash
kubectl get pods -l app=contador -o wide
```

Consulta los logs de cada Pod:

```bash
kubectl logs contador-0 --tail=5
kubectl logs contador-1 --tail=5
kubectl logs contador-2 --tail=5
```

### Preguntas

1. ¿Qué nombres tienen los Pods?
2. ¿Los nombres parecen aleatorios o siguen un patrón ordenado?
3. ¿Se ha creado algún ReplicaSet para este StatefulSet?

Compruébalo:

```bash
kubectl get replicasets
```

## Parte 2 - Eliminar un Pod del StatefulSet

Elimina uno de los Pods:

```bash
kubectl delete pod contador-1
```

Espera unos segundos y comprueba los Pods:

```bash
kubectl get pods -l app=contador -o wide
```

Consulta los logs del Pod recreado:

```bash
kubectl logs contador-1 --tail=5
```

### Preguntas

1. ¿Kubernetes ha vuelto a crear el Pod?
2. ¿El nuevo Pod conserva el mismo nombre?
3. ¿El contador de logs continúa donde estaba o empieza de nuevo?
4. ¿Por qué en este ejercicio se pierde el estado interno del contenedor?

## Parte 3 - Escalar el StatefulSet

Escala el StatefulSet a 4 réplicas:

```bash
kubectl scale statefulset/contador --replicas=4
```

Comprueba los Pods:

```bash
kubectl get pods -l app=contador -o wide
```

Ahora reduce a 2 réplicas:

```bash
kubectl scale statefulset/contador --replicas=2
```

Comprueba los Pods:

```bash
kubectl get pods -l app=contador -o wide
```

### Preguntas

1. ¿Qué Pod se crea al escalar de 3 a 4?
2. ¿Qué Pods desaparecen al reducir de 4 a 2?
3. ¿Qué diferencia observas respecto a un Deployment?

## Parte 4 - Crear un DaemonSet

Aplica el DaemonSet:

```bash
kubectl apply -f yaml/modulo5-ejercicio02-daemonset.yaml
```

Comprueba el DaemonSet:

```bash
kubectl get daemonset auditor-nodo
```

Comprueba los Pods y los nodos donde se ejecutan:

```bash
kubectl get pods -l app=auditor-nodo -o wide
```

Consulta logs de todos los Pods seleccionados por la etiqueta:

```bash
kubectl logs -l app=auditor-nodo --tail=10
```

Comprueba cuántos nodos hay:

```bash
kubectl get nodes
```

Comprueba si hay taints en los nodos:

```bash
kubectl describe nodes | grep -i taint -A1
```

### Preguntas

1. ¿Cuántos Pods ha creado el DaemonSet?
2. ¿Coincide con el número total de nodos?
3. Si no coincide, ¿qué nodo puede estar excluido y por qué?

## Parte 5 - DaemonSet con toleration para el control-plane

> Realiza esta parte solo si el DaemonSet anterior no se ejecuta en el nodo de control-plane por un taint `NoSchedule`.

Aplica el segundo DaemonSet:

```bash
kubectl apply -f yaml/modulo5-ejercicio02-daemonset-control-plane.yaml
```

Comprueba los Pods:

```bash
kubectl get pods -l app=auditor-nodo-control-plane -o wide
```

### Preguntas

1. ¿Se ha creado ahora un Pod también en el nodo de control-plane?
2. ¿Qué permite la sección `tolerations`?
3. ¿Por qué puede ser necesario en DaemonSets de monitorización o logging?

## Limpieza

Elimina los recursos creados:

```bash
kubectl delete -f yaml/modulo5-ejercicio02-daemonset-control-plane.yaml --ignore-not-found
kubectl delete -f yaml/modulo5-ejercicio02-daemonset.yaml --ignore-not-found
kubectl delete -f yaml/modulo5-ejercicio02-statefulset.yaml --ignore-not-found
```

Comprueba que no quedan Pods de estos ejercicios:

```bash
kubectl get pods -l app=contador
kubectl get pods -l app=auditor-nodo
kubectl get pods -l app=auditor-nodo-control-plane
```
