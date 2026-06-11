# Diagnóstico y recuperación de etcd después de una restauración fallida

## Contexto del problema

Durante el mantenimiento de `etcd`, se ha realizado una restauración de una snapshot en un clúster Kubernetes instalado con `kubeadm`.

Después de la restauración, el clúster deja de responder:

```bash
kubectl get nodes
```

Devuelve un error similar a:

```text
The connection to the server 10.0.0.100:6443 was refused - did you specify the right host or port?
```

Esto significa que el `kube-apiserver` no está disponible. Como `kube-apiserver` depende de `etcd`, el primer objetivo será comprobar si `etcd` está arrancando correctamente.

---

## 1. Comprobar el estado de etcd y del control plane

En el nodo master, ejecutamos:

```bash
sudo crictl ps -a | grep -E 'kube-apiserver|kube-controller-manager|kube-scheduler|etcd' || true
```

Una salida problemática puede ser parecida a esta:

```text
Exited   etcd
Exited   kube-apiserver
Running  kube-scheduler
Running  kube-controller-manager
```

Esto indica que:

* `etcd` está fallando.
* `kube-apiserver` también está fallando.
* `kube-scheduler` y `kube-controller-manager` pueden seguir en ejecución, pero no podrán trabajar correctamente sin el API Server.

---

## 2. Obtener el ID real del contenedor de etcd

No se debe ejecutar:

```bash
sudo crictl logs etcd
```

Ese comando falla porque `crictl logs` necesita el ID del contenedor, no el nombre lógico del contenedor.

Primero localizamos el contenedor de etcd:

```bash
sudo crictl ps -a --name etcd
```

Ejemplo de salida:

```text
CONTAINER      IMAGE          CREATED        STATE    NAME   ATTEMPT   POD ID       POD
a78ed6c34c7f   ee85eb1f0edd   2 minutes ago  Exited   etcd   6         3a2d794...   etcd-kubernetes-master
```

Ahora consultamos sus logs usando el ID real:

```bash
sudo crictl logs a78ed6c34c7f
```

---

## 3. Identificar el error de membresía de etcd

En los logs de etcd puede aparecer un error similar a:

```text
failed to update; member unknown
```

También pueden aparecer líneas como:

```text
Detected member only in v3store but missing in v2store
```

Este tipo de error indica que etcd no está arrancando correctamente por un problema de identidad o membresía interna tras la restauración de la snapshot.

En este caso, el problema se produjo porque la snapshot se restauró con una herramienta antigua o no adecuada para la versión real de etcd que usa Kubernetes.

---

## 4. Detener temporalmente el static Pod de etcd

En un clúster creado con `kubeadm`, etcd se ejecuta como static Pod definido en:

```bash
/etc/kubernetes/manifests/etcd.yaml
```

Para evitar que kubelet siga intentando arrancarlo en bucle, movemos temporalmente el manifiesto:

```bash
sudo mkdir -p /root/kubeadm-static-pods-backup

sudo mv /etc/kubernetes/manifests/etcd.yaml \
  /root/kubeadm-static-pods-backup/etcd.yaml
```

Esperamos unos segundos y comprobamos:

```bash
sleep 20
sudo crictl ps -a --name etcd
```

Si no aparece ningún contenedor nuevo de etcd, kubelet ya ha dejado de intentar arrancarlo.

---

## 5. Revisar los parámetros reales del manifiesto de etcd

Antes de restaurar de nuevo la snapshot, necesitamos conocer los parámetros exactos que kubeadm espera usar para etcd.

Ejecutamos:

```bash
sudo grep -E -- '--name=|--initial-cluster=|--initial-advertise-peer-urls=|--listen-peer-urls=|--advertise-client-urls=|--data-dir=' \
  /root/kubeadm-static-pods-backup/etcd.yaml
```

Ejemplo de salida:

```text
- --advertise-client-urls=https://10.0.0.100:2379
- --data-dir=/var/lib/etcd
- --initial-advertise-peer-urls=https://10.0.0.100:2380
- --initial-cluster=kubernetes-master=https://10.0.0.100:2380
- --listen-peer-urls=https://10.0.0.100:2380
- --name=kubernetes-master
```

En este caso, los valores importantes son:

```text
name: kubernetes-master
data-dir: /var/lib/etcd
initial-cluster: kubernetes-master=https://10.0.0.100:2380
initial-advertise-peer-urls: https://10.0.0.100:2380
```

---

## 6. Comprobar la versión de etcdctl instalada en el sistema

Es habitual que el sistema tenga instalado un `etcdctl` antiguo desde los repositorios de Ubuntu.

Comprobamos la versión:

```bash
which etcdctl
etcdctl --version
```

Ejemplo de salida problemática:

```text
/usr/bin/etcdctl
etcdctl version: 3.3.25
API version: 2
```

Este cliente es demasiado antiguo para restaurar correctamente una snapshot de un etcd moderno.

Ahora comprobamos la imagen de etcd que usa Kubernetes:

```bash
sudo crictl images | grep etcd
```

Ejemplo:

```text
registry.k8s.io/etcd   3.6.8-0   ee85eb1f0edd2   22.9MB
```

En este caso, Kubernetes está usando etcd 3.6.8.

Por tanto, no debemos restaurar la snapshot con `etcdctl 3.3.25`.

---

## 7. Confirmar que la imagen de etcd contiene las herramientas correctas

Comprobamos que la imagen contiene `etcdctl` moderno:

```bash
sudo ctr -n k8s.io run --rm \
  registry.k8s.io/etcd:3.6.8-0 \
  etcdctl-test \
  etcdctl version
```

Salida esperada:

```text
etcdctl version: 3.6.8
API version: 3.6
```

En etcd 3.6, la restauración de snapshots se realiza con `etcdutl`, no con `etcdctl snapshot restore`.

Comprobamos que `etcdutl` está disponible:

```bash
sudo ctr -n k8s.io run --rm \
  registry.k8s.io/etcd:3.6.8-0 \
  etcdutl-test \
  etcdutl version
```

Salida esperada:

```text
etcdutl version: 3.6.8
API version: 3.6
```

---

## 8. Apartar la restauración fallida

Antes de restaurar otra vez, movemos el directorio `/var/lib/etcd` actual, que contiene la restauración defectuosa.

No lo borramos directamente para conservarlo por si necesitamos analizarlo después:

```bash
sudo mv /var/lib/etcd \
  /var/lib/etcd-restauracion-fallida-$(date +%Y%m%d-%H%M%S)
```

Comprobamos:

```bash
sudo ls -ld /var/lib/etcd*
```

También verificamos que la snapshot existe:

```bash
sudo ls -lh /root/etcd-backup.db
```

---

## 9. Restaurar la snapshot con etcdutl desde la imagen correcta

Ahora restauramos la snapshot usando `etcdutl` desde la misma imagen de etcd que utiliza Kubernetes.

Ajustar la imagen si el clúster usa otra versión de etcd.

```bash
sudo ctr -n k8s.io run --rm \
  --mount type=bind,src=/root,dst=/backup,options=rbind:ro \
  --mount type=bind,src=/var/lib,dst=/varlib,options=rbind:rw \
  registry.k8s.io/etcd:3.6.8-0 \
  etcd-restore-368 \
  etcdutl snapshot restore /backup/etcd-backup.db \
    --data-dir=/varlib/etcd \
    --name=kubernetes-master \
    --initial-cluster=kubernetes-master=https://10.0.0.100:2380 \
    --initial-advertise-peer-urls=https://10.0.0.100:2380 \
    --initial-cluster-token=etcd-cluster-1
```

Una restauración correcta mostrará una salida similar a:

```text
restoring snapshot
Trimming membership information from the backend...
added member
restored snapshot
```

La línea:

```text
Trimming membership information from the backend...
```

es importante, porque indica que `etcdutl` está limpiando la información antigua de membresía antes de generar la nueva estructura del clúster etcd.

---

## 10. Comprobar que el directorio restaurado tiene la estructura correcta

Ejecutamos:

```bash
sudo ls -ld /var/lib/etcd
sudo find /var/lib/etcd -maxdepth 2 -type d -print
```

La salida debe incluir:

```text
/var/lib/etcd
/var/lib/etcd/member
/var/lib/etcd/member/snap
/var/lib/etcd/member/wal
```

---

## 11. Volver a activar etcd

Devolvemos el manifiesto estático de etcd a su ubicación original:

```bash
sudo mv /root/kubeadm-static-pods-backup/etcd.yaml \
  /etc/kubernetes/manifests/etcd.yaml
```

Esperamos unos segundos y comprobamos:

```bash
sleep 20
sudo crictl ps -a --name etcd
```

La salida esperada debe mostrar `etcd` en estado `Running`:

```text
CONTAINER      IMAGE          CREATED         STATE     NAME
ccee6dcb1bbc   ee85eb1f0edd   14 seconds ago Running   etcd
```

---

## 12. Comprobar si kube-apiserver se recupera

Ahora comprobamos el API Server:

```bash
sudo crictl ps -a --name kube-apiserver
```

Puede aparecer todavía en estado `Exited` o en `CrashLoopBackOff` porque falló varias veces mientras etcd estaba caído.

Consultamos sus logs si es necesario:

```bash
sudo crictl logs <ID_DEL_CONTENEDOR_KUBE_APISERVER>
```

Un error típico durante la recuperación puede ser:

```text
dial tcp 127.0.0.1:2379: connect: connection refused
```

Esto indica que el API Server intentó conectar con etcd cuando etcd todavía no estaba disponible.

---

## 13. Confirmar que etcd escucha en los puertos correctos

Ejecutamos:

```bash
sudo ss -lntp | grep -E '2379|2380|6443'
```

Antes de que el API Server esté recuperado, al menos deberíamos ver:

```text
127.0.0.1:2379    etcd
10.0.0.100:2379   etcd
10.0.0.100:2380   etcd
```

Esto confirma que etcd ya está escuchando correctamente.

---

## 14. Reiniciar kubelet si kube-apiserver queda en CrashLoopBackOff

Si `kube-apiserver` sigue sin recrearse porque kubelet lo tiene en backoff, se puede reiniciar kubelet:

```bash
sudo systemctl restart kubelet
```

Esperamos unos segundos:

```bash
sleep 20
sudo crictl ps -a --name kube-apiserver
```

La salida esperada debe mostrar el API Server en estado `Running`:

```text
CONTAINER      IMAGE          CREATED         STATE     NAME
d6b8b4a8240d   315157c5c4d7   24 seconds ago Running   kube-apiserver
```

---

## 15. Confirmar que el puerto 6443 vuelve a estar activo

Ejecutamos:

```bash
sudo ss -lntp | grep -E '2379|2380|6443'
```

La salida debe mostrar:

```text
127.0.0.1:2379    etcd
10.0.0.100:2379   etcd
10.0.0.100:2380   etcd
*:6443             kube-apiserver
```

Esto indica que tanto etcd como kube-apiserver están funcionando.

---

## 16. Validar el estado del clúster con kubectl

Finalmente, comprobamos el estado del clúster:

```bash
kubectl get nodes
```

Salida esperada:

```text
NAME                 STATUS   ROLES           AGE   VERSION
kubernetes-master    Ready    control-plane   3d    v1.36.1
kubernetes-worker1   Ready    <none>          3d    v1.36.1
kubernetes-worker2   Ready    <none>          3d    v1.36.1
```

También podemos comprobar los pods del sistema:

```bash
kubectl get pods -A
```

---

## Conclusión

El problema no estaba en la snapshot en sí, sino en cómo se había restaurado.

La restauración inicial se hizo con un `etcdctl` demasiado antiguo:

```text
etcdctl version: 3.3.25
API version: 2
```

Mientras que el clúster estaba usando:

```text
etcd version: 3.6.8
```

En versiones recientes de etcd, la restauración correcta debe hacerse con `etcdutl`, preferiblemente desde la misma imagen de etcd que utiliza Kubernetes:

```text
registry.k8s.io/etcd:3.6.8-0
```

Además, después de recuperar etcd, puede ser necesario reiniciar kubelet para que el `kube-apiserver` salga del estado de `CrashLoopBackOff` y vuelva a procesarse como static Pod.

---

## Comandos clave resumidos

```bash
# Ver control plane
sudo crictl ps -a | grep -E 'kube-apiserver|kube-controller-manager|kube-scheduler|etcd' || true

# Ver contenedor etcd
sudo crictl ps -a --name etcd

# Ver logs de etcd
sudo crictl logs <ID_CONTENEDOR_ETCD>

# Detener static Pod de etcd
sudo mkdir -p /root/kubeadm-static-pods-backup
sudo mv /etc/kubernetes/manifests/etcd.yaml /root/kubeadm-static-pods-backup/etcd.yaml

# Ver parámetros de etcd
sudo grep -E -- '--name=|--initial-cluster=|--initial-advertise-peer-urls=|--listen-peer-urls=|--advertise-client-urls=|--data-dir=' \
  /root/kubeadm-static-pods-backup/etcd.yaml

# Comprobar versión de etcdctl instalada
which etcdctl
etcdctl --version

# Ver imágenes etcd disponibles
sudo crictl images | grep etcd

# Comprobar etcdutl desde la imagen correcta
sudo ctr -n k8s.io run --rm \
  registry.k8s.io/etcd:3.6.8-0 \
  etcdutl-test \
  etcdutl version

# Apartar restauración fallida
sudo mv /var/lib/etcd /var/lib/etcd-restauracion-fallida-$(date +%Y%m%d-%H%M%S)

# Restaurar snapshot correctamente
sudo ctr -n k8s.io run --rm \
  --mount type=bind,src=/root,dst=/backup,options=rbind:ro \
  --mount type=bind,src=/var/lib,dst=/varlib,options=rbind:rw \
  registry.k8s.io/etcd:3.6.8-0 \
  etcd-restore-368 \
  etcdutl snapshot restore /backup/etcd-backup.db \
    --data-dir=/varlib/etcd \
    --name=kubernetes-master \
    --initial-cluster=kubernetes-master=https://10.0.0.100:2380 \
    --initial-advertise-peer-urls=https://10.0.0.100:2380 \
    --initial-cluster-token=etcd-cluster-1

# Reactivar etcd
sudo mv /root/kubeadm-static-pods-backup/etcd.yaml /etc/kubernetes/manifests/etcd.yaml

# Comprobar etcd
sleep 20
sudo crictl ps -a --name etcd

# Reiniciar kubelet si kube-apiserver está en CrashLoopBackOff
sudo systemctl restart kubelet

# Comprobar kube-apiserver
sleep 20
sudo crictl ps -a --name kube-apiserver

# Comprobar puertos
sudo ss -lntp | grep -E '2379|2380|6443'

# Validar clúster
kubectl get nodes
kubectl get pods -A
```
