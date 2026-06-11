# Laboratorio — Troubleshooting de un nodo worker `NotReady`

## Objetivo

Diagnosticar un fallo en uno de los nodos worker del clúster y recuperar su funcionamiento.

El clúster está formado por:

* 1 nodo control plane
* 2 nodos worker
* Kubernetes instalado con `kubeadm`
* Ubuntu 22.04
* Runtime de contenedores ya configurado

En este laboratorio hay un problema en uno de los workers.

## Escenario

Se ha desplegado una aplicación de prueba y un `DaemonSet` de comprobación llamado `node-canary`.

El equipo de operaciones informa de que:

* Uno de los workers no parece estar sano.
* Algunos pods no se comportan como se esperaba.
* El clúster sigue funcionando parcialmente porque todavía queda otro worker disponible.

Tu tarea es diagnosticar qué ocurre y recuperar el nodo afectado.

## Archivos iniciales

Aplica los manifiestos iniciales:

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 10-node-canary-daemonset.yaml
kubectl apply -f 20-demo-deployment.yaml
```

Comprueba el estado inicial:

```bash
kubectl get nodes -o wide
kubectl get pods -n cka-node-troubleshooting -o wide
```

## Tareas

### 1. Identificar el nodo afectado

Lista los nodos del clúster:

```bash
kubectl get nodes -o wide
```

Busca si algún nodo aparece en estado `NotReady`.

### 2. Revisar los pods del namespace del laboratorio

```bash
kubectl get pods -n cka-node-troubleshooting -o wide
```

Observa:

* En qué nodo está cada pod.
* Si algún pod aparece en estado anómalo.
* Si falta algún pod del `DaemonSet`.

### 3. Analizar el nodo afectado desde Kubernetes si es posible

Ejecuta:

```bash
kubectl describe node <NODO_AFECTADO>
```

Revisa especialmente:

* `Conditions`
* `Ready`
* `MemoryPressure`
* `DiskPressure`
* `PIDPressure`
* `NetworkUnavailable`
* `Events`

Preguntas guía:

* ¿El nodo está comunicándose con el control plane?
* ¿El problema parece de recursos?
* ¿El problema parece de red?
* ¿El problema parece del agente del nodo?


### 4. Recuperar el nodo

Aplica la corrección necesaria para que el nodo vuelva a `Ready`.

Después, verifica desde tu máquina de administración:

```bash
kubectl get nodes -o wide
kubectl get pods -n cka-node-troubleshooting -o wide
kubectl rollout status deploy/demo-web -n cka-node-troubleshooting
```

## Resultado esperado

Al finalizar:

* Todos los nodos deben aparecer como `Ready`.
* El `DaemonSet` debe tener un pod ejecutándose en cada worker.
* El `Deployment` debe estar disponible.
* Los pods deben poder ejecutarse de nuevo en el worker recuperado.

## Limpieza

Cuando termines:

```bash
kubectl delete ns cka-node-troubleshooting
```
