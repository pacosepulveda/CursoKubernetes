# Ejercicios Módulo 2

En estos ejercicios vamos a realizar tareas de diagnóstico de fallos y mantenimiento de un clúster Kubernetes instalado con `kubeadm`.

El entorno utilizado en el curso es:

- 3 instancias EC2 `t3.medium`.
- Ubuntu Server 22.04.
- Kubernetes instalado con `kubeadm`.
- `containerd` como Container Runtime Interface, CRI.
- Herramientas de diagnóstico: `kubectl`, `crictl`, `journalctl` y logs locales del nodo.

> **Importante:** estos ejercicios deben realizarse en el nodo `control-plane`, ya que vamos a modificar los manifiestos estáticos de los componentes del plano de control ubicados en `/etc/kubernetes/manifests`.

## Preparación: configurar `crictl`

Como utilizamos `containerd` como CRI, la herramienta `crictl` será muy útil para diagnosticar fallos de los componentes del clúster.

Primero instalamos `crictl`:

```bash
VERSION="v1.35.0"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz

sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin

rm crictl-$VERSION-linux-amd64.tar.gz
```

Configura `crictl` para que se comunique con `containerd`:

```bash
sudo tee /etc/crictl.yaml >/dev/null <<'EOF_CRIO'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF_CRIO
```

Comprueba que funciona:

```bash
sudo crictl info
sudo crictl ps
```

Se podría configurar crictl para no tener que usar `sudo`, pero no es recomendable porque implica dar permisos al usuario para comunicarse directamente con el runtime de contenedores.

Opcionalmente, si quieres usar el alias habitual del examen CKA:

```bash
alias k=kubectl
```

En estos ejercicios usaremos `kubectl` explícitamente para evitar confusiones.

## Ubicaciones útiles para diagnóstico

Cuando falle un componente del plano de control, especialmente el `kube-apiserver`, recuerda que `kubectl` puede dejar de funcionar porque necesita comunicarse con el API Server.

En esos casos, usa herramientas locales del nodo:

```bash
sudo crictl ps -a
sudo crictl pods -a
sudo journalctl -u kubelet -n 100 --no-pager
sudo journalctl -u kubelet -f
```

Logs habituales:

```bash
sudo ls /var/log/pods
sudo ls /var/log/containers
```

Para ver logs del `kube-apiserver` sin depender del nombre exacto del nodo ni del UID del Pod:

```bash
sudo tail -n 50 /var/log/pods/kube-system_kube-apiserver_*/kube-apiserver/*.log
```

Para ver logs del último contenedor del `kube-apiserver` con `crictl`:

```bash
CID=$(sudo crictl ps -a --name kube-apiserver -q | head -1)
sudo crictl logs "$CID"
```

> En este entorno no usaremos `docker ps` ni `docker logs`, ya que el runtime es `containerd`.

---

# Ejercicio 1: fallo en el API Server por argumento incorrecto

## Objetivo

Modificar el manifiesto estático del `kube-apiserver` añadiendo un argumento inexistente. Después, diagnosticar el fallo usando logs locales del nodo y restaurar el manifiesto original.

Debes sentirte cómodo con situaciones en las que el API Server no vuelve a arrancar y `kubectl` deja de responder.

## Pasos

Haz una copia de seguridad del manifiesto original:

```bash
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml ~/kube-apiserver.yaml.ori
```

Edita el manifiesto:

```bash
sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

Dentro de la lista `command:` del contenedor `kube-apiserver`, añade este argumento:

```yaml
    - --esto-esta-muy-mal
```

Debe quedar dentro de la lista de argumentos, por ejemplo:

```yaml
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=...
    - --esto-esta-muy-mal
```

Espera unos segundos y observa cómo kubelet intenta reiniciar el contenedor:

```bash
watch 'sudo crictl ps -a --name kube-apiserver'
```

Comprueba si `kubectl` responde:

```bash
kubectl -n kube-system get pods
```

Es probable que falle, porque el API Server no está disponible.

## Diagnóstico

Consulta el último contenedor del `kube-apiserver`:

```bash
sudo crictl ps -a --name kube-apiserver
```

Obtén sus logs:

```bash
CID=$(sudo crictl ps -a --name kube-apiserver -q | head -1)
sudo crictl logs "$CID"
```

También puedes consultar los logs en disco:

```bash
sudo tail -n 50 /var/log/pods/kube-system_kube-apiserver_*/kube-apiserver/*.log
```

Error esperado, o similar:

```text
Error: unknown flag: --esto-esta-muy-mal
```

También puedes revisar los logs del kubelet:

```bash
sudo journalctl -u kubelet -n 100 --no-pager
```

## Restauración

Restaura el manifiesto original:

```bash
sudo cp ~/kube-apiserver.yaml.ori /etc/kubernetes/manifests/kube-apiserver.yaml
```

Espera a que kubelet recree el contenedor correcto:

```bash
watch 'sudo crictl ps --name kube-apiserver'
```

Cuando el API Server vuelva a estar disponible, compruébalo con:

```bash
kubectl get nodes
kubectl -n kube-system get pods
```

---

# Ejercicio 2: configuración incorrecta de la conexión con etcd

## Objetivo

Modificar el argumento `--etcd-servers` del `kube-apiserver` para que apunte a un endpoint incorrecto. Después, diagnosticar el fallo sin depender de `/var/log/pods` ni `/var/log/containers`, usando principalmente `crictl` y `journalctl`.

## Pasos

Haz una copia de seguridad del manifiesto original:

```bash
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml ~/kube-apiserver.yaml.ori
```

Edita el manifiesto:

```bash
sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

Busca el argumento `--etcd-servers`. Normalmente será parecido a este:

```yaml
    - --etcd-servers=https://127.0.0.1:2379
```

Cámbialo por un endpoint incorrecto:

```yaml
    - --etcd-servers=https://127.0.0.1:12379
```

Guarda el fichero y espera a que kubelet reinicie el contenedor:

```bash
watch 'sudo crictl ps -a --name kube-apiserver'
```

Comprueba si `kubectl` responde:

```bash
kubectl get nodes
```

Es probable que falle, porque el API Server no puede conectarse correctamente con etcd.

## Diagnóstico sin usar `/var/log/pods` ni `/var/log/containers`

Consulta el contenedor del `kube-apiserver`:

```bash
sudo crictl ps -a --name kube-apiserver
```

Obtén sus logs:

```bash
CID=$(sudo crictl ps -a --name kube-apiserver -q | head -1)
sudo crictl logs "$CID"
```

Errores esperados, o similares:

```text
connection refused
```

```text
failed to connect to etcd
```

```text
Error while dialing
```

Revisa también los logs del kubelet:

```bash
sudo journalctl -u kubelet -n 100 --no-pager
```

O en tiempo real:

```bash
sudo journalctl -u kubelet -f
```

> En este ejercicio el manifiesto YAML es válido y el proceso del API Server puede llegar a arrancar, pero fallará al intentar conectarse con etcd.

## Restauración

Restaura el manifiesto original:

```bash
sudo cp ~/kube-apiserver.yaml.ori /etc/kubernetes/manifests/kube-apiserver.yaml
```

Espera a que el contenedor vuelva a levantarse:

```bash
watch 'sudo crictl ps --name kube-apiserver'
```

Comprueba el estado del clúster:

```bash
kubectl get nodes
kubectl -n kube-system get pods
```

---

# Ejercicio 3: YAML no válido en el manifiesto del API Server

## Objetivo

Modificar el manifiesto estático del `kube-apiserver` introduciendo YAML inválido. En este caso kubelet no podrá interpretar el fichero, por lo que probablemente ni siquiera llegará a crear un nuevo contenedor del API Server.

## Pasos

Haz una copia de seguridad del manifiesto original:

```bash
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml ~/kube-apiserver.yaml.ori
```

Edita el manifiesto:

```bash
sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

Introduce una línea claramente inválida al principio del fichero, por ejemplo:

```yaml
apiVersionESTO ESTA MUY MAL ::::: v1
```

Guarda el fichero.

Espera unos segundos y observa que puede no aparecer un nuevo contenedor del API Server:

```bash
watch 'sudo crictl ps -a --name kube-apiserver'
```

Comprueba si `kubectl` responde:

```bash
kubectl get nodes
```

Probablemente fallará porque el API Server no está disponible.

## Diagnóstico

En este caso, como el YAML no se puede parsear, kubelet puede no crear ningún contenedor nuevo. Por eso `crictl logs` puede no mostrar información útil.

Consulta los logs de kubelet:

```bash
sudo journalctl -u kubelet -n 100 --no-pager
```

O en tiempo real:

```bash
sudo journalctl -u kubelet -f
```

Filtra por el nombre del manifiesto:

```bash
sudo journalctl -u kubelet --since "10 minutes ago" | grep kube-apiserver
```

También puedes buscar errores relacionados con manifiestos estáticos:

```bash
sudo journalctl -u kubelet --since "10 minutes ago" | grep -i manifest
```

Error esperado, o similar:

```text
Could not process manifest file err="/etc/kubernetes/manifests/kube-apiserver.yaml: couldn't parse as pod"
```

o:

```text
yaml: mapping values are not allowed in this context
```

Comprueba que no hay logs nuevos útiles del contenedor:

```bash
sudo crictl ps -a --name kube-apiserver
```

> La diferencia clave con los ejercicios anteriores es que aquí el problema ocurre antes de que kubelet pueda crear el Pod estático correctamente.

## Restauración

Restaura el manifiesto original:

```bash
sudo cp ~/kube-apiserver.yaml.ori /etc/kubernetes/manifests/kube-apiserver.yaml
```

Espera a que kubelet procese de nuevo el manifiesto correcto:

```bash
watch 'sudo crictl ps --name kube-apiserver'
```

Cuando el API Server vuelva a estar disponible, comprueba el estado del clúster:

```bash
kubectl get nodes
kubectl -n kube-system get pods
```

---

# Limpieza final

Al terminar los ejercicios, asegúrate de que el manifiesto actual no contiene cambios de prueba:

```bash
sudo diff ~/kube-apiserver.yaml.ori /etc/kubernetes/manifests/kube-apiserver.yaml
```

Si hay diferencias inesperadas, restaura la copia original:

```bash
sudo cp ~/kube-apiserver.yaml.ori /etc/kubernetes/manifests/kube-apiserver.yaml
```

Comprueba que el clúster está operativo:

```bash
kubectl get nodes
kubectl -n kube-system get pods
```

Y que el API Server está ejecutándose:

```bash
sudo crictl ps --name kube-apiserver
```

---

# Notas finales

- Estos ejercicios deben ejecutarse solo en el nodo `control-plane`.
- Cuando el API Server falla, `kubectl` puede dejar de funcionar; esto es normal y forma parte del ejercicio.
- Para diagnosticar el plano de control caído, priorizar:
  - `sudo crictl ps -a --name kube-apiserver`
  - `sudo crictl logs <container-id>`
  - `sudo journalctl -u kubelet`
  - `/var/log/pods/kube-system_kube-apiserver_*/kube-apiserver/*.log`
- No usar `docker ps` ni `docker logs` en este entorno, porque el runtime es `containerd`.
- Evitar rutas hardcodeadas que incluyan el hostname del nodo o el UID del Pod.
- Si se usa el alias `k`, definirlo al principio del curso con `alias k=kubectl`.
