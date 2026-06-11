# Solución — Troubleshooting de un worker `NotReady`

## 1. Comprobar el estado de los nodos

```bash
kubectl get nodes -o wide
```

Salida esperada aproximada:

```text
NAME       STATUS     ROLES           AGE   VERSION
master-1   Ready      control-plane   ...   v1.36.1
worker-1   Ready      <none>          ...   v1.36.1
worker-2   NotReady   <none>          ...   v1.36.1
```

El síntoma principal es que uno de los workers aparece como `NotReady`.

## 2. Revisar los pods del laboratorio

```bash
kubectl get pods -n cka-node-troubleshooting -o wide
```

Podemos observar que los pods que estaban en el worker afectado pueden aparecer como `Running`, `Unknown`, `Terminating` o quedarse sin actualizar correctamente.

También puede verse que el `DaemonSet` no tiene todos los pods disponibles:

```bash
kubectl get daemonset -n cka-node-troubleshooting
```

Ejemplo:

```text
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
node-canary   2         2         1       2            1
```

Esto indica que uno de los workers no está operativo desde el punto de vista de Kubernetes.

## 3. Describir el nodo afectado

```bash
kubectl describe node worker-2
```

En la sección `Conditions`, el campo importante es `Ready`.

Podemos ver algo parecido a:

```text
Ready   Unknown   Kubelet stopped posting node status.
```

O bien:

```text
Ready   False     KubeletNotReady
```

Esto indica que el control plane ha dejado de recibir actualizaciones del kubelet de ese nodo.

## 4. Confirmar que no es un problema de scheduling

Este no es principalmente un problema de scheduler.

El scheduler puede seguir colocando pods en nodos sanos, pero el nodo `NotReady` no puede gestionar correctamente los pods porque el componente responsable de hablar con el control plane, crear pods y reportar estado es el `kubelet`.

Comprobación útil:

```bash
kubectl get pods -n cka-node-troubleshooting -o wide
kubectl get events -n cka-node-troubleshooting --sort-by='.lastTimestamp' | tail -n 20
```

## 5. Conectarse al worker afectado

Entramos por SSH al nodo afectado:

```bash
ssh ubuntu@<IP_DEL_WORKER_AFECTADO>
```

Una vez dentro, revisamos los servicios principales.

```bash
sudo systemctl status kubelet --no-pager
```

Salida esperada en este laboratorio:

```text
kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (...)
     Active: inactive (dead)
```

O bien:

```text
Active: failed
```

También conviene comprobar `containerd`:

```bash
sudo systemctl status containerd --no-pager
```

En este escenario, `containerd` debería estar activo. Eso ayuda a acotar el problema al `kubelet`.

## 6. Revisar logs del kubelet

```bash
sudo journalctl -u kubelet -n 80 --no-pager
```

Si el servicio está parado, puede que no haya errores complejos. El punto importante es confirmar que el kubelet no está ejecutándose.

## 7. Aplicar el fix

Arrancamos el kubelet:

```bash
sudo systemctl start kubelet
```

Si estaba deshabilitado:

```bash
sudo systemctl enable kubelet
sudo systemctl start kubelet
```

Comprobamos el estado:

```bash
sudo systemctl status kubelet --no-pager
```

Debe aparecer:

```text
Active: active (running)
```

## 8. Verificar desde Kubernetes

Volvemos a la máquina desde la que usamos `kubectl` y ejecutamos:

```bash
kubectl get nodes -o wide
```

El worker debe volver a `Ready`:

```text
NAME       STATUS   ROLES           AGE   VERSION
master-1   Ready    control-plane   ...   v1.36.1
worker-1   Ready    <none>          ...   v1.36.1
worker-2   Ready    <none>          ...   v1.36.1
```

Verificamos los pods:

```bash
kubectl get pods -n cka-node-troubleshooting -o wide
kubectl get daemonset -n cka-node-troubleshooting
kubectl rollout status deploy/demo-web -n cka-node-troubleshooting
```

El `DaemonSet` debería tener todos sus pods disponibles:

```text
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
node-canary   2         2         2       2            2
```

## 9. Conclusión

La causa raíz era que el servicio `kubelet` estaba parado en uno de los workers.

El kubelet es el agente de Kubernetes que corre en cada nodo. Sus responsabilidades incluyen:

* Registrar el nodo en el clúster.
* Reportar el estado del nodo.
* Crear y supervisar pods.
* Comunicar el estado de contenedores al control plane.

Cuando el kubelet deja de ejecutarse, el nodo deja de enviar heartbeats y Kubernetes lo marca como `NotReady`.

## Comandos clave del diagnóstico

```bash
kubectl get nodes -o wide
kubectl describe node <NODO_AFECTADO>
kubectl get pods -A -o wide
kubectl get events -A --sort-by='.lastTimestamp' | tail -n 30
sudo systemctl status kubelet --no-pager
sudo journalctl -u kubelet -n 80 --no-pager
sudo systemctl start kubelet
```
