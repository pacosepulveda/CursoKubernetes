# Instalación de Kubernetes con kubeadm en Ubuntu 22.04

Esta guía describe cómo instalar un clúster Kubernetes de tres nodos usando `kubeadm` en instancias Ubuntu 22.04.

El laboratorio está pensado para un entorno con tres nodos:

| Nodo | Hostname | IP privada |
|---|---|---:|
| Nodo master / control-plane | `kubernetes-master` | `10.0.0.100` |
| Worker 1 | `kubernetes-worker1` | `10.0.0.101` |
| Worker 2 | `kubernetes-worker2` | `10.0.0.102` |

El objetivo es instalar Kubernetes usando `containerd` como runtime de contenedores y Calico como plugin de red.

> **Nota:** En esta guía se instala Kubernetes desde el repositorio `v1.35` para poder realizar posteriormente una práctica de actualización a una versión superior.

---

## 1. Limpieza previa si una instalación anterior ha fallado

Este apartado solo debe ejecutarse si ya se había intentado instalar Kubernetes previamente y el clúster ha quedado en un estado incorrecto.

En ese caso, ejecuta los siguientes comandos en el nodo afectado:

```bash
sudo kubeadm reset -f

sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni/
sudo rm -rf /var/lib/kubelet/*
sudo rm -rf /etc/kubernetes/
sudo rm -rf ~/.kube/

sudo apt-get purge kubeadm kubectl kubelet kubernetes-cni kube* -y --allow-change-held-packages

sudo apt-get autoremove -y

sudo rm -rf /var/lib/etcd
```

### ¿Qué estamos eliminando?

`kubeadm reset` revierte buena parte de la configuración generada durante `kubeadm init` o `kubeadm join`, pero no elimina absolutamente todos los restos del clúster. Por eso se borran manualmente directorios relacionados con:

- configuración de red CNI;
- datos locales de `kubelet`;
- manifiestos y certificados de Kubernetes;
- configuración local de `kubectl`;
- datos de `etcd`, en el caso del nodo master.

Este paso ayuda a evitar errores provocados por configuraciones antiguas, certificados previos o restos de una red CNI mal desplegada.

---

## 2. Preparación común en todos los nodos

Los pasos de esta sección deben ejecutarse en los tres nodos:

- `kubernetes-master`
- `kubernetes-worker1`
- `kubernetes-worker2`

---

## 2.1. Desactivar swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### ¿Por qué hay que desactivar swap?

Kubernetes necesita tener un control predecible sobre la memoria disponible en cada nodo. Si el sistema operativo empieza a mover memoria a swap, `kubelet` puede tomar decisiones incorrectas sobre la presión de memoria real del nodo.

Por este motivo, en una instalación clásica con `kubeadm`, se desactiva swap antes de inicializar el clúster.

El primer comando desactiva swap en la sesión actual. El segundo comenta las entradas de swap en `/etc/fstab` para evitar que swap vuelva a activarse automáticamente tras reiniciar la máquina.

---

## 2.2. Cargar módulos del kernel necesarios

```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

### ¿Para qué sirven estos módulos?

El módulo `overlay` permite usar el sistema de ficheros OverlayFS, habitual en runtimes de contenedores.

El módulo `br_netfilter` permite que el tráfico que pasa por bridges de Linux pueda ser procesado por las reglas de filtrado de red del kernel. Esto es importante para que Kubernetes y los plugins CNI puedan gestionar correctamente el tráfico de red de los pods.

El fichero `/etc/modules-load.d/containerd.conf` hace que estos módulos se carguen automáticamente en futuros reinicios.

---

## 2.3. Configurar parámetros de red del kernel

```bash
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

### ¿Por qué son necesarios estos parámetros?

Estos parámetros permiten que el tráfico de red de los pods sea procesado correctamente por el sistema:

- `net.bridge.bridge-nf-call-iptables = 1` permite que el tráfico IPv4 que atraviesa bridges Linux pase por `iptables`.
- `net.bridge.bridge-nf-call-ip6tables = 1` hace lo mismo para IPv6.
- `net.ipv4.ip_forward = 1` permite que el nodo reenvíe tráfico IPv4 entre interfaces, algo necesario para que los pods puedan comunicarse fuera de su propio namespace de red.

Después de ejecutar `sudo sysctl --system`, se puede comprobar que los valores se han aplicado con:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

El resultado esperado es:

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

---

## 2.4. Instalar containerd

```bash
sudo apt-get update
sudo apt-get install -y containerd
```

Kubernetes necesita un runtime de contenedores compatible con CRI. En este laboratorio se utiliza `containerd`, que es una opción habitual y ligera para ejecutar contenedores en nodos Kubernetes.

---

## 2.5. Crear la configuración de containerd

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

### ¿Por qué recreamos el archivo `config.toml`?

Al instalar `containerd`, puede que no exista un fichero de configuración completo en `/etc/containerd/config.toml`, o que la configuración por defecto del paquete no incluya todos los parámetros que queremos revisar.

Con este comando generamos una configuración explícita y completa a partir de los valores por defecto de `containerd`. Esto hace que el laboratorio sea más predecible y facilita modificar parámetros importantes, como el driver de cgroups.

---

## 2.6. Configurar el driver de cgroups de containerd

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

### ¿Por qué usamos `SystemdCgroup = true`?

Kubernetes recomienda que el runtime de contenedores y `kubelet` usen el mismo driver de cgroups. En distribuciones modernas como Ubuntu 22.04, lo habitual es usar `systemd` como gestor de cgroups.

Al establecer `SystemdCgroup = true`, hacemos que `containerd` use `systemd` para la gestión de cgroups. Esto evita inconsistencias entre `kubelet` y el runtime de contenedores.

---

## 2.7. Reiniciar y habilitar containerd

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

Con estos comandos aplicamos la nueva configuración y dejamos `containerd` habilitado para que arranque automáticamente si la máquina se reinicia.

Puedes comprobar el estado del servicio con:

```bash
systemctl status containerd
```

---

## 2.8. Añadir el repositorio de Kubernetes

En este laboratorio instalaremos Kubernetes desde la rama estable `v1.35`.

```bash
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### ¿Por qué usamos una versión anterior?

En un curso de administración de Kubernetes es útil instalar una versión que no sea la última disponible para poder practicar después el proceso de actualización del clúster.

El objetivo es partir de una versión estable anterior y actualizar posteriormente a una versión superior siguiendo el procedimiento correcto con `kubeadm`.

---

## 2.9. Instalar kubelet, kubeadm y kubectl

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Componentes instalados

- `kubeadm`: herramienta para inicializar y unir nodos al clúster.
- `kubelet`: agente que se ejecuta en cada nodo y se encarga de gestionar los pods.
- `kubectl`: cliente de línea de comandos para interactuar con la API de Kubernetes.

El comando `apt-mark hold` evita que estos paquetes se actualicen automáticamente con un `apt upgrade`. En Kubernetes, las actualizaciones deben realizarse de forma controlada y siguiendo un orden concreto.

Puedes comprobar las versiones instaladas con:

```bash
kubeadm version
kubectl version --client
kubelet --version
```

---

## 3. Configurar el nombre de cada nodo

Ejecuta el comando correspondiente en cada instancia.

En el nodo master:

```bash
sudo hostnamectl set-hostname kubernetes-master
```

En el worker 1:

```bash
sudo hostnamectl set-hostname kubernetes-worker1
```

En el worker 2:

```bash
sudo hostnamectl set-hostname kubernetes-worker2
```

Para aplicar el cambio en la shell actual sin cerrar la sesión SSH, puedes ejecutar:

```bash
exec bash
```

Comprueba el hostname con:

```bash
hostname
```

---

## 4. Inicializar el clúster en el nodo master

Este apartado debe ejecutarse únicamente en el nodo master.

```bash
sudo kubeadm init --apiserver-advertise-address=10.0.0.100 --pod-network-cidr=192.168.0.0/16
```

### Explicación del comando

- `--apiserver-advertise-address=10.0.0.100` indica la IP privada por la que el API Server de Kubernetes será anunciado al resto de nodos.
- `--pod-network-cidr=192.168.0.0/16` define el rango de red que se usará para las IPs de los pods. Este valor debe ser coherente con el plugin de red que se instalará después.

En este laboratorio se usará Calico, por eso se utiliza `192.168.0.0/16`.

---

## 5. Configurar kubectl en el nodo master

Después de inicializar el clúster, configura `kubectl` para el usuario actual:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### ¿Qué estamos haciendo?

`kubeadm init` genera el fichero `/etc/kubernetes/admin.conf`, que contiene las credenciales de administración del clúster.

Copiamos ese fichero a `$HOME/.kube/config` para que el usuario actual pueda ejecutar comandos `kubectl` sin tener que indicar manualmente el fichero de configuración en cada comando.

---

## 6. Instalar Calico como plugin de red

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

### ¿Por qué necesitamos un plugin de red?

Después de ejecutar `kubeadm init`, el clúster todavía no tiene una red de pods funcional. Kubernetes delega esa parte en un plugin CNI.

Calico proporciona conectividad entre pods y permite trabajar posteriormente con políticas de red (`NetworkPolicy`), lo que resulta especialmente interesante en un curso de administración de Kubernetes.

---

## 7. Comprobar el estado de los pods del sistema

```bash
watch kubectl get pods -A
```

Espera hasta que los pods del sistema estén en estado `Running` o `Completed`.

En otra terminal, también puedes revisar los nodos:

```bash
kubectl get nodes
```

Al principio solo aparecerá el nodo master. Debes esperar a que esté en estado `Ready` antes de unir los workers.

---

## 8. Activar autocompletado de kubectl

En el nodo master:

```bash
sudo apt-get install bash-completion
```

Añade el autocompletado de `kubectl` al fichero `.bashrc`:

```bash
echo 'source <(kubectl completion bash)' >>~/.bashrc
```

Aplica los cambios en la sesión actual:

```bash
source ~/.bashrc
```

A partir de este momento podrás usar tabulación para completar comandos y recursos de Kubernetes.

---

## 9. Generar el comando para unir los workers

En el nodo master, ejecuta:

```bash
kubeadm token create --print-join-command
```

El comando devolverá una salida similar a esta:

```bash
sudo kubeadm join 10.0.0.100:6443 --token XXX \
  --discovery-token-ca-cert-hash sha256:XXXX
```

Copia el comando generado. Será necesario ejecutarlo en cada nodo worker.

> **Importante:** No reutilices tokens antiguos ni comandos de otros laboratorios. Cada clúster genera su propio token y su propio hash de descubrimiento.

---

## 10. Unir los nodos worker al clúster

Este paso debe ejecutarse en cada worker.

En `kubernetes-worker1` y `kubernetes-worker2`, ejecuta el comando generado en el nodo master:

```bash
sudo kubeadm join 10.0.0.100:6443 --token XXX \
  --discovery-token-ca-cert-hash sha256:XXXX
```

Sustituye `XXX` y `XXXX` por los valores reales generados por tu clúster.

---

## 11. Comprobar el estado final del clúster

Vuelve al nodo master y ejecuta:

```bash
kubectl get nodes -o wide
```

El resultado esperado es que los tres nodos aparezcan en estado `Ready`:

```text
NAME                 STATUS   ROLES           AGE   VERSION   INTERNAL-IP
kubernetes-master    Ready    control-plane   ...   v1.35.x   10.0.0.100
kubernetes-worker1   Ready    <none>          ...   v1.35.x   10.0.0.101
kubernetes-worker2   Ready    <none>          ...   v1.35.x   10.0.0.102
```

También puedes comprobar los pods del sistema:

```bash
kubectl get pods -A
```

---

## 12. Comandos útiles de comprobación

Ver información del clúster:

```bash
kubectl cluster-info
```

Ver nodos:

```bash
kubectl get nodes -o wide
```

Ver pods de todos los namespaces:

```bash
kubectl get pods -A
```

Ver eventos recientes:

```bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Ver estado de `kubelet` en un nodo:

```bash
systemctl status kubelet
```

Ver logs de `kubelet`:

```bash
journalctl -u kubelet -f
```

---

## 13. Resumen del flujo completo

1. Preparar todos los nodos.
2. Desactivar swap.
3. Cargar módulos del kernel.
4. Configurar parámetros de red.
5. Instalar y configurar `containerd`.
6. Añadir el repositorio de Kubernetes `v1.35`.
7. Instalar `kubelet`, `kubeadm` y `kubectl`.
8. Configurar hostnames.
9. Ejecutar `kubeadm init` en el master.
10. Instalar Calico.
11. Generar el comando `kubeadm join`.
12. Unir los workers.
13. Validar que los tres nodos están `Ready`.
