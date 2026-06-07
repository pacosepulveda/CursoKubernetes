# Ejercicios Módulo 9 - Actualización, etcd y monitorización

> Entorno previsto: clúster Kubernetes instalado con `kubeadm` sobre 3 instancias EC2 Ubuntu 22.04, runtime `containerd`, CNI Calico y un nodo de plano de control.
>
> Estos ejercicios incluyen operaciones delicadas. No los ejecutes en un clúster compartido o de producción sin backup y ventana de mantenimiento.

## 1. Actualizar el clúster con kubeadm

En este ejercicio actualizaremos el clúster de Kubernetes **de la versión 1.35.x a la versión 1.36.x**.

Kubernetes no soporta saltarse versiones minor en una actualización con `kubeadm`. Es decir, se puede actualizar de `1.35.x` a `1.36.x`, pero no de `1.34.x` directamente a `1.36.x`.

> Ejecuta los pasos de plano de control en el nodo control-plane. Los pasos de worker se ejecutan de uno en uno en cada nodo worker.

### 1.1 Comprobar el estado inicial

En el nodo control-plane:

```bash
kubectl get nodes -o wide
kubectl version
kubeadm version
```

Comprueba también qué versión minor está usando el clúster:

```bash
kubectl get nodes
```

La columna `VERSION` debería mostrar una versión `v1.35.x` antes de empezar este ejercicio.

### 1.2 Configurar el repositorio APT de Kubernetes 1.36

Si el clúster se instaló con paquetes de `pkgs.k8s.io`, cada versión minor usa su propio repositorio APT. Para actualizar de 1.35 a 1.36 hay que cambiar el repositorio de paquetes.

En **todos los nodos**:

```bash
sudo sed -i 's#core:/stable:/v1.35/deb/#core:/stable:/v1.36/deb/#' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
```

Comprueba las versiones disponibles:

```bash
apt-cache madison kubeadm
```

Elige la última versión patch disponible de `1.36.x`. En los comandos siguientes se usa una variable para evitar hardcodear el número exacto.

```bash
export K8S_VERSION=$(apt-cache madison kubeadm | awk '/1\.36\./ {print $3; exit}')
echo "$K8S_VERSION"
```

Si la variable queda vacía, revisa el repositorio APT antes de continuar.

## 2. Actualizar el nodo control-plane

### 2.1 Actualizar kubeadm

En el nodo control-plane:

```bash
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=${K8S_VERSION}
sudo apt-mark hold kubeadm
```

Comprueba la versión:

```bash
kubeadm version
```

### 2.2 Ver el plan de actualización

```bash
sudo kubeadm upgrade plan
```

Revisa que el plan proponga actualizar a una versión `v1.36.x`.

### 2.3 Aplicar la actualización del plano de control

Obtén la versión de Kubernetes a partir del paquete Debian instalado:

```bash
export K8S_UPGRADE_VERSION="v$(echo ${K8S_VERSION} | sed 's/-.*//')"
echo "$K8S_UPGRADE_VERSION"
```

Aplica la actualización:

```bash
sudo kubeadm upgrade apply ${K8S_UPGRADE_VERSION}
```

### 2.4 Drenar el nodo control-plane

Obtén el nombre real del nodo control-plane:

```bash
export CONTROL_PLANE_NODE=$(kubectl get nodes -l node-role.kubernetes.io/control-plane -o jsonpath='{.items[0].metadata.name}')
echo "$CONTROL_PLANE_NODE"
```

### 2.5 Actualizar kubelet y kubectl

En el nodo control-plane:

```bash
sudo apt-mark unhold kubelet kubectl
sudo apt-get update
sudo apt-get install -y kubelet=${K8S_VERSION} kubectl=${K8S_VERSION}
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Comprueba el estado:

```bash
kubectl get nodes
```

## 3. Actualizar los nodos worker

Repite este bloque **worker por worker**. No actualices todos los workers a la vez.

### 3.1 Elegir un worker

Desde el control-plane:

```bash
kubectl get nodes
```

Elige uno de los workers y guarda su nombre:

```bash
export WORKER_NODE=<nombre-del-worker>
```

### 3.2 Actualizar kubeadm en el worker

Conéctate por SSH al worker elegido y ejecuta:

```bash
sudo sed -i 's#core:/stable:/v1.35/deb/#core:/stable:/v1.36/deb/#' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
export K8S_VERSION=$(apt-cache madison kubeadm | awk '/1\.36\./ {print $3; exit}')
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=${K8S_VERSION}
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node
```

### 3.3 Drenar el worker

Desde el control-plane:

```bash
kubectl drain "$WORKER_NODE" --ignore-daemonsets --delete-emptydir-data
```

### 3.4 Actualizar kubelet y kubectl en el worker

En el worker:

```bash
sudo apt-mark unhold kubelet kubectl
sudo apt-get update
sudo apt-get install -y kubelet=${K8S_VERSION} kubectl=${K8S_VERSION}
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### 3.5 Desacordonar el worker

Desde el control-plane:

```bash
kubectl uncordon "$WORKER_NODE"
kubectl get nodes
```

Repite el proceso con el resto de workers.

### 3.6 Verificación final

Cuando todos los nodos estén actualizados:

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl version
```

Todos los nodos deberían estar `Ready` y en versión `v1.36.x`.

## 4. Mantenimiento de etcd

En un clúster kubeadm con un solo nodo de plano de control, etcd suele ejecutarse como un **static Pod** definido en:

```bash
/etc/kubernetes/manifests/etcd.yaml
```

> Importante: la restauración de etcd es una operación destructiva. Hazla únicamente en un laboratorio. En producción se requiere un procedimiento de recuperación ante desastres validado.

### 4.1 Localizar el Pod de etcd

```bash
kubectl -n kube-system get pods -l component=etcd -o wide
```

Guarda el nombre del Pod:

```bash
export ETCD_POD=$(kubectl -n kube-system get pods -l component=etcd -o jsonpath='{.items[0].metadata.name}')
echo "$ETCD_POD"
```

Inspecciona el Pod:

```bash
kubectl -n kube-system describe pod "$ETCD_POD"
```

Fíjate en los certificados y parámetros:

- `--cert-file`
- `--key-file`
- `--trusted-ca-file`
- `--advertise-client-urls`
- `--listen-client-urls`

### 4.2 Comprobar miembros y estado de etcd desde el Pod

```bash
kubectl -n kube-system exec "$ETCD_POD" -- sh -c '
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --write-out=table
'
```

```bash
kubectl -n kube-system exec "$ETCD_POD" -- sh -c '
ETCDCTL_API=3 etcdctl endpoint status \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --write-out=table
'
```

### 4.3 Instalar `etcdctl` en el nodo control-plane

Puedes ejecutar `etcdctl` desde el Pod de etcd o instalarlo en el host. Para instalarlo en Ubuntu:

```bash
sudo apt-get update
sudo apt-get install -y etcd-client
etcdctl version
```

Si el paquete no está disponible en tu repositorio Ubuntu, usa el binario oficial de la versión de etcd que corresponda a tu clúster.

### 4.4 Crear un snapshot de etcd

En el nodo control-plane:

```bash
sudo ETCDCTL_API=3 etcdctl snapshot save /root/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

Verifica el snapshot:

```bash
sudo ETCDCTL_API=3 etcdctl snapshot status /root/etcd-backup.db --write-out=table
```

### 4.5 Crear un recurso de prueba después del backup

```bash
kubectl apply -f modulo9-test-pod.yaml
kubectl get pod modulo9-test
```

El objetivo es comprobar que este Pod desaparece después de restaurar el snapshot anterior.

## 5. Restaurar etcd desde snapshot

> Advertencia: este procedimiento restaura el estado del clúster al momento del snapshot. Los objetos creados después del backup desaparecerán.

### 5.1 Detener temporalmente los static Pods del plano de control

En el nodo control-plane:

```bash
sudo mkdir -p /root/kubeadm-static-pods-backup
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /root/kubeadm-static-pods-backup/
sudo mv /etc/kubernetes/manifests/kube-controller-manager.yaml /root/kubeadm-static-pods-backup/
sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /root/kubeadm-static-pods-backup/
sudo mv /etc/kubernetes/manifests/etcd.yaml /root/kubeadm-static-pods-backup/
```

Espera a que kubelet detenga los componentes:

```bash
sudo crictl ps -a | grep -E 'kube-apiserver|kube-controller-manager|kube-scheduler|etcd' || true
```

### 5.2 Restaurar el snapshot en un nuevo directorio

No restaures encima de `/var/lib/etcd` directamente. Primero mueve la base de datos actual:

```bash
sudo mv /var/lib/etcd /var/lib/etcd.old.$(date +%Y%m%d%H%M%S)
```

Restaura el snapshot:

```bash
sudo ETCDCTL_API=3 etcdctl snapshot restore /root/etcd-backup.db \
  --data-dir=/var/lib/etcd
```

Asegura permisos:

```bash
sudo chown -R root:root /var/lib/etcd
```

### 5.3 Arrancar de nuevo el plano de control

Devuelve los manifiestos estáticos:

```bash
sudo mv /root/kubeadm-static-pods-backup/*.yaml /etc/kubernetes/manifests/
```

Espera uno o dos minutos y comprueba:

```bash
kubectl get nodes
kubectl get pods -A
```

Comprueba que el Pod creado después del backup ya no existe:

```bash
kubectl get pod modulo9-test
```

Debería aparecer `NotFound`.

## 6. Monitorización: Kubernetes Dashboard

Kubernetes Dashboard no se despliega por defecto. Actualmente la instalación recomendada es mediante Helm.

> Dashboard es útil para demos, pero no debe exponerse públicamente. No lo cambies a `LoadBalancer` en este laboratorio. Accede mediante `kubectl port-forward` o `kubectl proxy`.

### 6.1 Instalar Helm si no está instalado

```bash
helm version
```

Si no existe Helm, instálalo siguiendo el procedimiento oficial para Ubuntu o el método que se haya usado en el curso.

### 6.2 Instalar Kubernetes Dashboard

```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm repo update
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
  --create-namespace \
  --namespace kubernetes-dashboard
```

Comprueba los recursos:

```bash
kubectl -n kubernetes-dashboard get pods,svc
```

### 6.3 Crear usuario admin para laboratorio

Aplica el manifiesto local:

```bash
kubectl apply -f modulo9-dashboard-admin.yaml
```

Crea un token temporal:

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

Guarda el token solo para esta sesión de laboratorio.

### 6.4 Acceder al Dashboard

Comprueba el Service creado por el chart:

```bash
kubectl -n kubernetes-dashboard get svc
```

En versiones actuales del chart, el acceso suele pasar por el servicio `kubernetes-dashboard-kong-proxy`.

Usa port-forward:

```bash
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
```

Abre en tu navegador local:

```text
https://localhost:8443
```

El navegador mostrará una advertencia por certificado autofirmado. Acepta la advertencia solo en este entorno de laboratorio.

## 7. Logging y monitorización en EKS

El bloque de EKS/CloudWatch no forma parte del laboratorio kubeadm en EC2. Si se imparte también una práctica específica de EKS, debe ir en un módulo separado y sin credenciales embebidas.

No incluyas usuarios, contraseñas ni URLs de consola en el material entregado a alumnos.

Para un clúster kubeadm sobre EC2, las prácticas recomendadas de este curso deberían centrarse en:

- `kubectl logs`
- logs de kubelet con `journalctl -u kubelet`
- logs de Pods en `/var/log/pods`
- logs de contenedores con `crictl logs`
- métricas con `metrics-server` y `kubectl top`
- observabilidad adicional con Prometheus/Grafana o Fluent Bit si se añade como módulo específico

## 8. Limpieza opcional

Eliminar Dashboard:

```bash
helm uninstall kubernetes-dashboard -n kubernetes-dashboard
kubectl delete namespace kubernetes-dashboard
```

Eliminar el Pod de prueba si sigue existiendo:

```bash
kubectl delete pod modulo9-test --ignore-not-found
```
