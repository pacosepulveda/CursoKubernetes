# Ejercicios Módulo 7 - Volúmenes, Secrets y ConfigMaps

El objetivo de este módulo es practicar almacenamiento temporal, almacenamiento persistente, Secrets y ConfigMaps en Kubernetes.

Estos ejercicios están pensados para un clúster Kubernetes instalado con `kubeadm` en 3 instancias EC2 con Ubuntu 22.04 y containerd. No dependen de manifiestos remotos.

> Nota sobre almacenamiento: Kubernetes no crea almacenamiento persistente real por sí solo. Para persistencia hace falta un `PersistentVolume` respaldado por algún sistema de almacenamiento: disco local, NFS, EBS con CSI driver, etc. En este módulo usaremos almacenamiento local y, como ejercicio adicional, NFS.

## Preparación

Comprueba los nodos disponibles:

```bash
kubectl get nodes -o wide
```

Para algunos ejercicios necesitaremos saber el nombre de un nodo worker:

```bash
kubectl get nodes
```

---

## 1. Volumen `emptyDir`

`emptyDir` es un volumen temporal que se crea vacío cuando se crea un Pod. Su contenido se conserva mientras exista el Pod, incluso si se reinicia un contenedor dentro del Pod. Cuando el Pod se elimina, también se elimina el contenido del `emptyDir`.

Por defecto, `emptyDir` usa el almacenamiento efímero del nodo. Solo se crea en memoria si se configura explícitamente:

```yaml
emptyDir:
  medium: Memory
```

En este ejercicio usaremos un `emptyDir` normal, compartido por tres contenedores del mismo Pod.

Aplica el manifiesto:

```bash
kubectl apply -f modulo7-emptydir.yaml
```

Comprueba el Pod:

```bash
kubectl get pod myvolumes-pod
```

Escribe un archivo desde el primer contenedor:

```bash
kubectl exec -it myvolumes-pod -c container-1 -- sh
```

Dentro del contenedor:

```sh
echo "archivo creado desde container-1" > /data/container1.txt
ls -l /data
exit
```

Comprueba el mismo archivo desde el segundo contenedor, que monta el mismo volumen en otra ruta:

```bash
kubectl exec -it myvolumes-pod -c container-2 -- sh
```

Dentro del contenedor:

```sh
ls -l /cache
cat /cache/container1.txt
exit
```

Comprueba también desde el tercer contenedor:

```bash
kubectl exec -it myvolumes-pod -c container-3 -- sh -c 'ls -l /shared && cat /shared/container1.txt'
```

Elimina y recrea el Pod para comprobar que el contenido desaparece:

```bash
kubectl delete pod myvolumes-pod
kubectl apply -f modulo7-emptydir.yaml
kubectl exec -it myvolumes-pod -c container-1 -- ls -l /data
```

Limpieza:

```bash
kubectl delete -f modulo7-emptydir.yaml
```

---

## 2. PersistentVolume local

En este ejercicio crearemos un `PersistentVolume` local en uno de los nodos worker. Esto sirve para entender PV/PVC, pero tiene una limitación importante: el Pod que consuma el volumen deberá ejecutarse en el mismo nodo donde existe la ruta local.

### 2.1 Elegir un nodo worker

Elige uno de los workers y guarda su nombre en una variable. Sustituye el valor por el nombre real de tu nodo:

```bash
export WORKER_NODE=<nombre-del-worker>
```

Ejemplo:

```bash
export WORKER_NODE=ip-10-0-1-25.eu-west-1.compute.internal
```

### 2.2 Preparar el directorio en el worker

Ejecuta en el worker elegido:

```bash
sudo mkdir -p /mnt/volumes/pv-local-1
sudo chmod 777 /mnt/volumes/pv-local-1
```

> En un entorno de producción no usaríamos permisos `777`. Aquí se hace solo para simplificar el laboratorio.

### 2.3 Ajustar el YAML del PV

Edita `modulo7-pv-local.yaml` y sustituye `REEMPLAZAR_POR_NOMBRE_DEL_WORKER` por el nombre real del nodo:

```bash
sed -i "s/REEMPLAZAR_POR_NOMBRE_DEL_WORKER/${WORKER_NODE}/" modulo7-pv-local.yaml
```

Aplica el PV y el PVC:

```bash
kubectl apply -f modulo7-pv-local.yaml
kubectl apply -f modulo7-pvc-local.yaml
```

Comprueba el estado:

```bash
kubectl get pv
kubectl get pvc
```

El PVC debe quedar en estado `Bound`.

### 2.4 Crear un Pod que usa el PVC

Aplica el Pod:

```bash
kubectl apply -f modulo7-pod-local-pvc.yaml
```

Comprueba dónde se ha planificado:

```bash
kubectl get pod pod-local-pvc -o wide
```

El Pod debería ejecutarse en el nodo indicado en el `nodeAffinity` del PV.

Escribe un archivo desde el Pod:

```bash
kubectl exec -it pod-local-pvc -- sh -c 'echo "dato persistente local" > /usr/share/nginx/html/index.html && cat /usr/share/nginx/html/index.html'
```

Comprueba el archivo en el worker:

```bash
sudo cat /mnt/volumes/pv-local-1/index.html
```

Elimina y recrea el Pod:

```bash
kubectl delete pod pod-local-pvc
kubectl apply -f modulo7-pod-local-pvc.yaml
kubectl exec pod-local-pvc -- cat /usr/share/nginx/html/index.html
```

El dato se mantiene porque el volumen local no se ha eliminado.

Limpieza:

```bash
kubectl delete -f modulo7-pod-local-pvc.yaml
kubectl delete -f modulo7-pvc-local.yaml
kubectl delete -f modulo7-pv-local.yaml
```

Si el PV queda en estado `Released`, puedes eliminarlo manualmente y limpiar el directorio en el worker:

```bash
sudo rm -rf /mnt/volumes/pv-local-1/*
```

---

## 3. Secrets

Los `Secret` permiten almacenar información sensible y consumirla desde Pods como variables de entorno o como archivos montados en un volumen.

Puntos importantes:

- Los Secrets son objetos namespaced.
- Pueden consumirse como variables de entorno o como volúmenes.
- En los nodos, los datos montados se exponen normalmente usando `tmpfs`.
- El límite de tamaño de un Secret es 1 MiB.
- Por defecto, Kubernetes almacena Secrets en etcd codificados en base64, no cifrados. Para producción se recomienda cifrado de Secrets en reposo.

Crea un Secret desde un literal:

```bash
kubectl create secret generic apikey --from-literal=apikey=A19fh68B001j
```

Comprueba sus metadatos:

```bash
kubectl describe secret apikey
```

El valor no se muestra con `describe`. Para verlo codificado:

```bash
kubectl get secret apikey -o yaml
```

Para decodificarlo:

```bash
kubectl get secret apikey -o jsonpath='{.data.apikey}' | base64 -d; echo
```

Crea un Pod que consuma el Secret como volumen y como variable de entorno:

```bash
kubectl apply -f modulo7-secret-pod.yaml
```

Comprueba el valor montado como archivo:

```bash
kubectl exec -it consumesec -- sh -c 'ls -l /etc/apikey && cat /etc/apikey/apikey'
```

Comprueba el valor como variable de entorno:

```bash
kubectl exec -it consumesec -- sh -c 'echo $APIKEY'
```

Limpieza:

```bash
kubectl delete -f modulo7-secret-pod.yaml
kubectl delete secret apikey
```

---

## 4. ConfigMaps

Los `ConfigMap` permiten separar configuración no sensible de las imágenes de contenedor.

### 4.1 Crear ConfigMaps desde literales

```bash
kubectl create configmap config --from-literal=foo=lala --from-literal=foo2=lolo
kubectl get configmap config -o yaml
kubectl describe configmap config
```

### 4.2 Crear un ConfigMap desde un archivo

```bash
printf 'foo3=lili\nfoo4=lele\n' > config.txt
kubectl create configmap configmap2 --from-file=config.txt
kubectl get configmap configmap2 -o yaml
```

### 4.3 Usar una clave de ConfigMap como variable de entorno

Crea el ConfigMap:

```bash
kubectl create configmap options --from-literal=var5=val5
```

Aplica el Pod:

```bash
kubectl apply -f modulo7-configmap-env-key.yaml
```

Comprueba la variable:

```bash
kubectl exec -it nginx-cm-key -- env | grep '^option='
```

Resultado esperado:

```text
option=val5
```

### 4.4 Cargar todas las claves de un ConfigMap como variables de entorno

Crea el ConfigMap:

```bash
kubectl create configmap anotherone --from-literal=var6=val6 --from-literal=var7=val7
```

Aplica el Pod:

```bash
kubectl apply -f modulo7-configmap-envfrom.yaml
```

Comprueba las variables:

```bash
kubectl exec -it nginx-cm-envfrom -- env | grep '^var[67]='
```

> En el YAML original de este ejercicio había un error: el campo correcto es `name: anotherone`, no `cname: anotherone`.

### 4.5 Montar un ConfigMap como volumen

Crea el ConfigMap:

```bash
kubectl create configmap cmvolume --from-literal=var8=val8 --from-literal=var9=val9
```

Aplica el Pod:

```bash
kubectl apply -f modulo7-configmap-volume.yaml
```

Comprueba los archivos montados:

```bash
kubectl exec -it nginx-cm-volume -- sh -c 'ls -l /etc/lala && cat /etc/lala/var8 && echo && cat /etc/lala/var9'
```

Limpieza de ConfigMaps y Pods:

```bash
kubectl delete pod nginx-cm-key nginx-cm-envfrom nginx-cm-volume --ignore-not-found
kubectl delete configmap config configmap2 options anotherone cmvolume --ignore-not-found
rm -f config.txt
```

---

## 5. Ejercicio de PersistentVolume usando NFS

Este ejercicio usa el nodo control-plane como servidor NFS y los workers como clientes. Es útil para entender `ReadWriteMany`, pero no es una arquitectura recomendada para producción: el control-plane no debería actuar como servidor de almacenamiento de aplicaciones.

### 5.1 Identificar la IP privada del nodo control-plane

En el nodo control-plane:

```bash
hostname -I
```

Guarda la IP privada en una variable en tu terminal del control-plane:

```bash
export NFS_SERVER_IP=<ip-privada-del-control-plane>
```

Ejemplo:

```bash
export NFS_SERVER_IP=10.0.1.10
```

### 5.2 Instalar y configurar el servidor NFS en el control-plane

Ejecuta en el nodo control-plane:

```bash
sudo apt update
sudo apt install -y nfs-kernel-server

sudo mkdir -p /srv/nfs/pv1
sudo mkdir -p /srv/nfs/pv2
sudo chmod -R 777 /srv/nfs
```

Configura los exports. Ajusta el CIDR `10.0.0.0/16` si tu VPC usa otro rango:

```bash
sudo tee /etc/exports > /dev/null <<'EXPORTS'
/srv/nfs/pv1 10.0.0.0/16(rw,sync,no_subtree_check,no_root_squash)
/srv/nfs/pv2 10.0.0.0/16(rw,sync,no_subtree_check,no_root_squash)
EXPORTS

sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
sudo exportfs -v
```

En AWS, asegúrate de que los Security Groups permiten tráfico NFS entre nodos. Como mínimo, TCP/2049 desde los workers hacia el control-plane.

### 5.3 Preparar los workers como clientes NFS

Ejecuta en cada worker:

```bash
sudo apt update
sudo apt install -y nfs-common
showmount -e <ip-privada-del-control-plane>
```

Ejemplo:

```bash
showmount -e 10.0.1.10
```

### 5.4 Ajustar los YAML de NFS

Sustituye el placeholder por la IP privada del servidor NFS:

```bash
sed -i "s/REEMPLAZAR_POR_IP_NFS/${NFS_SERVER_IP}/g" modulo7-pv-nfs.yaml
```

Aplica los PVs:

```bash
kubectl apply -f modulo7-pv-nfs.yaml
kubectl get pv
```

Deberías ver dos PVs en estado `Available`.

Aplica el PVC:

```bash
kubectl apply -f modulo7-pvc-nfs.yaml
kubectl get pvc
kubectl get pv
```

El PVC debe quedar en estado `Bound`.

### 5.5 Crear un Pod con el PVC NFS

```bash
kubectl apply -f modulo7-pod-nfs.yaml
kubectl get pod pod-nfs-demo -o wide
```

Escribe un archivo desde el Pod:

```bash
kubectl exec -it pod-nfs-demo -- sh -c 'echo "<h1>Hola desde NFS</h1>" > /usr/share/nginx/html/index.html'
```

Comprueba el archivo en el servidor NFS:

```bash
cat /srv/nfs/pv1/index.html
```

También puedes leerlo desde el propio contenedor:

```bash
kubectl exec pod-nfs-demo -- cat /usr/share/nginx/html/index.html
```

### 5.6 Demostración de persistencia

Elimina el Pod y créalo de nuevo:

```bash
kubectl delete pod pod-nfs-demo
kubectl apply -f modulo7-pod-nfs.yaml
kubectl exec pod-nfs-demo -- cat /usr/share/nginx/html/index.html
```

El archivo debe seguir existiendo.

### 5.7 Múltiples Pods compartiendo el mismo volumen

NFS permite `ReadWriteMany`, por lo que varios Pods pueden montar el mismo PVC simultáneamente.

```bash
kubectl apply -f modulo7-deployment-nfs.yaml
kubectl get pods -l app=nginx-nfs -o wide
```

Escribe desde uno de los Pods:

```bash
POD=$(kubectl get pod -l app=nginx-nfs -o jsonpath='{.items[0].metadata.name}')
kubectl exec "$POD" -- sh -c 'echo "Compartido por NFS" > /usr/share/nginx/html/shared.txt'
```

Comprueba desde todos los Pods:

```bash
for p in $(kubectl get pod -l app=nginx-nfs -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'); do
  echo "--- $p"
  kubectl exec "$p" -- cat /usr/share/nginx/html/shared.txt
done
```

Limpieza:

```bash
kubectl delete -f modulo7-deployment-nfs.yaml --ignore-not-found
kubectl delete -f modulo7-pod-nfs.yaml --ignore-not-found
kubectl delete -f modulo7-pvc-nfs.yaml --ignore-not-found
kubectl delete -f modulo7-pv-nfs.yaml --ignore-not-found
```

Si quieres limpiar los datos en el servidor NFS:

```bash
sudo rm -rf /srv/nfs/pv1/* /srv/nfs/pv2/*
```

---

## Resumen de errores corregidos respecto al material original

- Se han eliminado dependencias de YAML remotos en S3 y GitHub.
- `emptyDir` no se crea en memoria por defecto; se crea en disco efímero del nodo salvo que se indique `medium: Memory`.
- Los ejemplos de ConfigMap ahora usan nombres de Pod distintos para evitar conflictos con Pods `nginx` existentes.
- En el ejemplo de `envFrom`, el campo correcto es `name`, no `cname`.
- El ejercicio de PV local ahora incluye `nodeAffinity`, necesario para volúmenes locales.
- El ejercicio NFS ya no asume que la IP del servidor es `10.0.0.100` ni que la VPC es `/24`.
- Se ha añadido advertencia sobre Security Groups de AWS para NFS.
