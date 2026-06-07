# Ejercicios Módulo 3: redes en Kubernetes con Calico

El objetivo de estos ejercicios es comprobar el funcionamiento de la red de pods en un clúster Kubernetes instalado con `kubeadm` en instancias EC2 y usando Calico como CNI.

En estos ejercicios comprobaremos:

- El estado de los nodos y de Calico.
- La comunicación entre pods en el mismo nodo.
- La comunicación entre pods en nodos diferentes.
- La salida de los pods hacia Internet.
- La IP del pod y su tabla de rutas.
- Algunos recursos de Calico instalados en el clúster.

> Nota para AWS EC2: si Calico está usando rutas nativas o encapsulación entre subredes, es habitual necesitar deshabilitar el `source/destination check` de las instancias EC2. Si los pods de nodos distintos no se comunican entre sí, revisad este punto, las reglas del Security Group y la configuración del IPPool de Calico.

## Archivos necesarios

Para estos ejercicios se usa el archivo:

```bash
modulo3-network-debug.yaml
```

El manifiesto crea un namespace `demo-redes` y un Deployment con 6 pods de diagnóstico basados en `nicolaka/netshoot:v0.15`.

Esta imagen incluye herramientas como `ping`, `ip`, `ifconfig`, `curl`, `dig`, `traceroute`, `tcpdump` y `calicoctl`, por lo que es más adecuada para ejercicios de red que una imagen mínima tipo `nginx` o `busybox`.

## 1. Comprobar el estado inicial del clúster

Comprobamos que los nodos están `Ready`:

```bash
kubectl get nodes -o wide
```

Salida esperada aproximada:

```text
NAME          STATUS   ROLES           AGE   VERSION   INTERNAL-IP
controlplane Ready    control-plane    ...   v1.35.x   10.0.x.x
worker1      Ready    <none>           ...   v1.35.x   10.0.x.x
worker2      Ready    <none>           ...   v1.35.x   10.0.x.x
```

Comprobamos que Calico está funcionando:

```bash
kubectl get pods -n calico-system -o wide
```

Si Calico se instaló con manifiestos clásicos en vez de operador, puede estar en `kube-system`:

```bash
kubectl get pods -n kube-system -o wide | grep -i calico
```

Deberíamos ver pods como `calico-node`, `calico-kube-controllers` o componentes equivalentes en estado `Running`.

## 2. Crear los pods de diagnóstico

Creamos los recursos del ejercicio:

```bash
kubectl apply -f modulo3-network-debug.yaml
```

Salida esperada:

```text
namespace/demo-redes created
deployment.apps/network-debug created
```

Verificamos que los pods están en ejecución:

```bash
kubectl get pods -n demo-redes -o wide
```

Salida esperada aproximada:

```text
NAME                             READY   STATUS    RESTARTS   AGE   IP              NODE
network-debug-xxxxxxxxxx-aaaaa   1/1     Running   0          30s   172.16.x.x      worker1
network-debug-xxxxxxxxxx-bbbbb   1/1     Running   0          30s   172.16.x.x      worker2
network-debug-xxxxxxxxxx-ccccc   1/1     Running   0          30s   172.16.x.x      worker1
...
```

> Las IPs y los nombres de nodo no serán iguales a los de esta salida. Dependen del rango de pods configurado en Calico y de los nombres reales de las instancias EC2.

Cuando instalamos Kubernetes con `kubeadm`, probablemente indicaste un rango de red para los Pods, por ejemplo:

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Después Calico toma ese rango global y lo divide internamente en bloques más pequeños, normalmente /26 en IPv4.

```bash
Rango global de Pods:
192.168.0.0/16

Calico lo divide en bloques:
192.168.227.64/26    → asignado a worker1
192.168.227.128/26   → asignado a worker2
otro bloque /26      → podría estar asignado al master
otros bloques /26    → disponibles
```

Calico hace esto para que cada nodo pueda asignar IPs a sus propios Pods localmente, sin tener que pedir una IP al clúster entero cada vez que se crea un Pod.

Podemos comprobar el rango global de Calico:

```bash
kubectl get ippools.crd.projectcalico.org
kubectl get ippools.crd.projectcalico.org default-ipv4-ippool -o yaml
```

También podemos ver los bloques que tiene asignado cada nodo:

```bash
kubectl get blockaffinities.crd.projectcalico.org
```

O con más detalle:

```bash
kubectl get blockaffinities.crd.projectcalico.org -o yaml
```

## 3. Identificar pods en el mismo nodo y en nodos distintos

Guardamos la lista de pods con sus IPs y nodos:

```bash
kubectl get pods -n demo-redes -o wide
```

Para seleccionar automáticamente un pod origen:

```bash
SRC_POD=$(kubectl get pods -n demo-redes -l app=network-debug \
  -o jsonpath='{.items[0].metadata.name}')

SRC_NODE=$(kubectl get pod -n demo-redes "$SRC_POD" \
  -o jsonpath='{.spec.nodeName}')

SRC_IP=$(kubectl get pod -n demo-redes "$SRC_POD" \
  -o jsonpath='{.status.podIP}')

echo "Pod origen: $SRC_POD"
echo "Nodo origen: $SRC_NODE"
echo "IP origen: $SRC_IP"
```

Buscamos otro pod que esté en el mismo nodo:

```bash
SAME_NODE_POD=$(kubectl get pods -n demo-redes -l app=network-debug \
  --field-selector spec.nodeName="$SRC_NODE" \
  -o jsonpath='{.items[?(@.metadata.name!="'"$SRC_POD"'")].metadata.name}' \
  | awk '{print $1}')

SAME_NODE_IP=$(kubectl get pod -n demo-redes "$SAME_NODE_POD" \
  -o jsonpath='{.status.podIP}')

echo "Pod en el mismo nodo: $SAME_NODE_POD"
echo "IP del pod en el mismo nodo: $SAME_NODE_IP"
```

Buscamos un pod que esté en otro nodo:

```bash
OTHER_NODE_POD=$(kubectl get pods -n demo-redes -l app=network-debug \
  -o jsonpath='{range .items[?(@.spec.nodeName!="'"$SRC_NODE"'")]}{.metadata.name}{"\n"}{end}' \
  | head -n 1)

OTHER_NODE_IP=$(kubectl get pod -n demo-redes "$OTHER_NODE_POD" \
  -o jsonpath='{.status.podIP}')

OTHER_NODE=$(kubectl get pod -n demo-redes "$OTHER_NODE_POD" \
  -o jsonpath='{.spec.nodeName}')

echo "Pod en otro nodo: $OTHER_NODE_POD"
echo "Nodo remoto: $OTHER_NODE"
echo "IP del pod en otro nodo: $OTHER_NODE_IP"
```

Si alguna variable sale vacía, revisa que haya varios pods en `Running` y que estén repartidos entre al menos dos nodos:

```bash
kubectl get pods -n demo-redes -o wide
```

## 4. Comunicación pod a pod en el mismo nodo

Ejecutamos `ping` desde el pod origen hacia otro pod que está en el mismo nodo:

```bash
kubectl exec -n demo-redes -it "$SRC_POD" -- ping -c 4 "$SAME_NODE_IP"
```

Salida esperada aproximada:

```text
PING 172.16.x.x (172.16.x.x): 56 data bytes
64 bytes from 172.16.x.x: seq=0 ttl=63 time=0.xxx ms
64 bytes from 172.16.x.x: seq=1 ttl=63 time=0.xxx ms
64 bytes from 172.16.x.x: seq=2 ttl=63 time=0.xxx ms
64 bytes from 172.16.x.x: seq=3 ttl=63 time=0.xxx ms

--- 172.16.x.x ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
```


También podemos acceder de forma interactiva al pod:

```bash
kubectl exec -n demo-redes -it "$SRC_POD" -- bash
```

Si falla, revisa:

```bash
kubectl get pods -n demo-redes -o wide
kubectl describe pod -n demo-redes "$SRC_POD"
kubectl get pods -A | grep -i calico
```

## 5. Comunicación pod a pod entre nodos diferentes

Ejecutamos `ping` desde el pod origen hacia un pod que está en otro nodo:

```bash
kubectl exec -n demo-redes -it "$SRC_POD" -- ping -c 4 "$OTHER_NODE_IP"
```

Salida esperada aproximada:

```text
PING 172.16.x.x (172.16.x.x): 56 data bytes
64 bytes from 172.16.x.x: seq=0 ttl=62 time=1.xxx ms
64 bytes from 172.16.x.x: seq=1 ttl=62 time=1.xxx ms
64 bytes from 172.16.x.x: seq=2 ttl=62 time=1.xxx ms
64 bytes from 172.16.x.x: seq=3 ttl=62 time=1.xxx ms

--- 172.16.x.x ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
```

Si este ping falla pero el ping entre pods del mismo nodo funciona, el problema suele estar en la comunicación entre nodos. En EC2 revisa especialmente:

- `source/destination check` deshabilitado en las instancias.
- Reglas del Security Group entre los nodos.
- Configuración del IPPool de Calico.
- Modo de encapsulación de Calico: IPIP, VXLAN o rutas nativas.

Comandos útiles:

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide | grep -i calico
kubectl logs -n calico-system -l k8s-app=calico-node --tail=50
```

Si Calico está en `kube-system`:

```bash
kubectl logs -n kube-system -l k8s-app=calico-node --tail=50
```

## 6. Comunicación externa desde los pods

Probamos resolución DNS y salida HTTP:

```bash
kubectl exec -n demo-redes -it "$SRC_POD" -- dig kubernetes.io
kubectl exec -n demo-redes -it "$SRC_POD" -- curl -I https://kubernetes.io
```

También podemos probar `ping`, aunque algunas redes o destinos pueden bloquear ICMP:

```bash
kubectl exec -n demo-redes -it "$SRC_POD" -- ping -c 4 8.8.8.8
kubectl exec -n demo-redes -it "$SRC_POD" -- ping -c 4 www.google.com
```

Si DNS falla:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```

Si DNS funciona pero no hay salida a Internet, revisa `natOutgoing` en el IPPool de Calico y la salida a Internet de las instancias EC2.

## 7. Explorar la IP del pod

Entramos en el pod origen:

```bash
kubectl exec -n demo-redes -it "$SRC_POD" -- sh
```

Dentro del pod, ejecutamos:

```bash
ip addr show
```

También podemos usar:

```bash
ifconfig
```

Salida esperada aproximada:

```text
eth0@if...: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu ...
    inet 172.16.x.x/32 scope global eth0
lo: <LOOPBACK,UP,LOWER_UP>
    inet 127.0.0.1/8 scope host lo
```

La IP debe coincidir con la columna `IP` de:

```bash
kubectl get pod -n demo-redes "$SRC_POD" -o wide
```

## 8. Explorar la ruta por defecto del pod

Dentro del pod, ejecutamos:

```bash
ip route
```

Salida esperada aproximada con Calico:

```text
default via 169.254.1.1 dev eth0
169.254.1.1 dev eth0 scope link
```

La puerta de enlace `169.254.1.1` es habitual en pods gestionados por Calico. No debe confundirse con una IP real de la VPC.

Salimos del pod:

```bash
exit
```

## 9. Inspeccionar recursos de Calico

Si Calico se instaló con el operador de Tigera, podemos revisar los recursos principales:

```bash
kubectl get installation.operator.tigera.io default -o yaml
kubectl get ippool.crd.projectcalico.org -o wide
```

Para ver el detalle del IPPool:

```bash
kubectl get ippool.crd.projectcalico.org -o yaml
```

Busca campos como:

```yaml
spec:
  cidr: ...
  ipipMode: ...
  vxlanMode: ...
  natOutgoing: true
```

Si `natOutgoing` no está en `true`, los pods pueden tener problemas para salir a Internet si sus IPs no son enrutables fuera del clúster.

## 10. Limpieza

Cuando termines el ejercicio:

```bash
kubectl delete -f modulo3-network-debug.yaml
```

Comprobamos que no quedan pods del ejercicio:

```bash
kubectl get pods -n demo-redes
```

El namespace debería haberse eliminado junto con los pods:

```bash
kubectl get namespace demo-redes
```

## Resumen de problemas frecuentes

| Síntoma | Causa probable | Comprobación |
|---|---|---|
| Los pods no pasan de `ContainerCreating` | CNI no está funcionando | `kubectl get pods -A | grep -i calico` |
| Pod a pod en el mismo nodo funciona, pero entre nodos falla | Problema entre nodos, encapsulación, Security Group o source/destination check | `kubectl get pods -n demo-redes -o wide` |
| DNS falla desde el pod | CoreDNS no está funcionando o red hacia CoreDNS rota | `kubectl get pods -n kube-system -l k8s-app=kube-dns` |
| DNS funciona, pero no hay salida a Internet | Falta NAT o salida desde las EC2 | Revisar IPPool `natOutgoing` y rutas/NAT Gateway/Internet Gateway |
| `ping www.google.com` falla, pero `curl https://kubernetes.io` funciona | ICMP bloqueado | Usar `curl` o `dig` como prueba principal |

