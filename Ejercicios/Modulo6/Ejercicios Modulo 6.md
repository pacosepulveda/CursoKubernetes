# Ejercicios Módulo 6: Servicios, DNS y exposición de aplicaciones

## Objetivo

En este módulo vamos a trabajar con los objetos `Service` de Kubernetes para entender:

- Por qué no conviene depender directamente de las IPs de los Pods.
- Cómo exponer Pods dentro del clúster con un Service `ClusterIP`.
- Cómo funciona el descubrimiento de servicios mediante DNS.
- Cómo exponer una aplicación fuera del clúster usando `NodePort`.
- Qué ocurre con un Service `LoadBalancer` en un clúster kubeadm sobre EC2.

> Entorno esperado: clúster Kubernetes instalado con kubeadm sobre instancias EC2, runtime `containerd`, CNI Calico y CoreDNS funcionando.

---

## 0. Comprobaciones iniciales

Comprobamos que los nodos están listos:

```bash
kubectl get nodes -o wide
```

Comprobamos que CoreDNS está funcionando:

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system get svc kube-dns
```

Salida orientativa:

```text
NAME                       READY   STATUS    RESTARTS   AGE   IP              NODE
coredns-xxxxxxxxxx-abcde   1/1     Running   0          10m   10.244.x.y      node-1
coredns-xxxxxxxxxx-fghij   1/1     Running   0          10m   10.244.x.z      node-2
```

> Si CoreDNS no está en `Running`, los ejercicios de DNS no funcionarán correctamente.

---

## 1. Crear el namespace y el Deployment de nginx

Aplicamos el manifiesto del Deployment:

```bash
kubectl apply -f modulo6-nginx.yaml
```

El manifiesto crea:

- Namespace `my-nginx`.
- Deployment `my-nginx`.
- 2 réplicas de nginx.

Comprobamos los Pods:

```bash
kubectl -n my-nginx get pods -o wide
```

Salida orientativa:

```text
NAME                        READY   STATUS    RESTARTS   AGE   IP              NODE
my-nginx-xxxxxxxxxx-aaaaa   1/1     Running   0          20s   10.244.1.10     worker-1
my-nginx-xxxxxxxxxx-bbbbb   1/1     Running   0          20s   10.244.2.15     worker-2
```

Las IPs reales dependerán de Calico y de la red de Pods configurada en vuestro clúster.

Podemos ver solo las IPs de los Pods con:

```bash
kubectl -n my-nginx get pods -l app=my-nginx -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\t"}{.spec.nodeName}{"\n"}{end}'
```

---

## 2. Probar acceso directo a las IPs de los Pods

Las IPs de los Pods son accesibles dentro del clúster, pero **no son estables**. Si un Pod se elimina y se recrea, normalmente recibirá otra IP.

Creamos un Pod cliente para hacer pruebas:

```bash
kubectl apply -f modulo6-client.yaml
kubectl -n my-nginx wait --for=condition=Ready pod/client --timeout=120s
```

Guardamos la IP de uno de los Pods nginx:

```bash
export NGINX_POD_IP=$(kubectl -n my-nginx get pods -l app=my-nginx -o jsonpath='{.items[0].status.podIP}')
echo $NGINX_POD_IP
```

Desde el Pod cliente, hacemos una petición directa al Pod nginx:

```bash
kubectl -n my-nginx exec client -- curl -sI http://$NGINX_POD_IP
```

Salida esperada orientativa:

```text
HTTP/1.1 200 OK
Server: nginx/1.27.x
```

Ahora eliminamos un Pod nginx:

```bash
kubectl -n my-nginx delete pod -l app=my-nginx --field-selector=status.phase=Running --wait=false
```

Esperamos a que el Deployment vuelva a tener 2 réplicas disponibles:

```bash
kubectl -n my-nginx rollout status deployment/my-nginx
kubectl -n my-nginx get pods -o wide
```

Comprueba que las IPs pueden haber cambiado. Este es el problema que resuelve un `Service`: proporcionar una dirección estable y un nombre DNS estable para un conjunto dinámico de Pods.

---

## 3. Crear un Service ClusterIP

Aplicamos el Service `ClusterIP`:

```bash
kubectl apply -f modulo6-service-clusterip.yaml
```

Comprobamos el Service:

```bash
kubectl -n my-nginx get svc my-nginx
```

Salida orientativa:

```text
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
my-nginx   ClusterIP   10.96.123.45    <none>        80/TCP    10s
```

La `CLUSTER-IP` es estable mientras exista el Service.

Comprobamos a qué Pods apunta el Service:

```bash
kubectl -n my-nginx get endpoints my-nginx
```

En Kubernetes recientes también se puede consultar `EndpointSlice`, que es el mecanismo recomendado internamente:

```bash
kubectl -n my-nginx get endpointslices -l kubernetes.io/service-name=my-nginx
kubectl -n my-nginx describe svc my-nginx
```

El Service selecciona los Pods que tienen la etiqueta:

```text
app=my-nginx
```

Puedes comprobarlo con:

```bash
kubectl -n my-nginx get pods -l app=my-nginx --show-labels
```

---

## 4. Acceder al Service desde otro Pod

Probamos acceso por nombre corto, porque el cliente está en el mismo namespace:

```bash
kubectl -n my-nginx exec client -- curl -sI http://my-nginx
```

Salida esperada orientativa:

```text
HTTP/1.1 200 OK
Server: nginx/1.27.x
```

También podemos usar el FQDN completo del Service:

```bash
kubectl -n my-nginx exec client -- curl -sI http://my-nginx.my-nginx.svc.cluster.local
```

El patrón general del FQDN de un Service es:

```text
<service>.<namespace>.svc.cluster.local
```

---

## 5. DNS de Services

La imagen `curlimages/curl` no incluye siempre herramientas como `nslookup`, pero sí podemos probar la resolución usando `curl`.

Para usar herramientas DNS, lanzamos un Pod temporal con `busybox`:

```bash
kubectl -n my-nginx run dns-test --rm -it --restart=Never --image=busybox:1.36 -- nslookup my-nginx
```

Salida orientativa:

```text
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      my-nginx
Address 1: 10.96.123.45 my-nginx.my-nginx.svc.cluster.local
```

Probamos ahora el FQDN:

```bash
kubectl -n my-nginx run dns-test-fqdn --rm -it --restart=Never --image=busybox:1.36 -- nslookup my-nginx.my-nginx.svc.cluster.local
```

---

## 6. Variables de entorno de Service

Kubernetes puede inyectar variables de entorno sobre Services existentes cuando se crea un Pod. Esta forma de descubrimiento existe por compatibilidad, pero en la práctica se prefiere DNS.

El Pod `client` se creó antes del Service, así que puede que no tenga variables para `MY_NGINX`:

```bash
kubectl -n my-nginx exec client -- env | grep MY_NGINX || true
```

Recreamos el Pod cliente para que se cree después del Service:

```bash
kubectl -n my-nginx delete pod client
kubectl apply -f modulo6-client.yaml
kubectl -n my-nginx wait --for=condition=Ready pod/client --timeout=120s
```

Volvemos a comprobar:

```bash
kubectl -n my-nginx exec client -- env | grep MY_NGINX
```

Salida esperada orientativa:

```text
MY_NGINX_SERVICE_HOST=10.96.123.45
MY_NGINX_SERVICE_PORT=80
```

> Aunque estas variables existen, para comunicación entre aplicaciones dentro del clúster es mejor usar DNS: `http://my-nginx` o `http://my-nginx.my-nginx.svc.cluster.local`.

---

## 7. Exponer el Service con NodePort

`NodePort` permite acceder a un Service desde fuera del clúster usando la IP de cualquier nodo y un puerto del rango `30000-32767`.

Aplicamos el Service `NodePort`:

```bash
kubectl apply -f modulo6-service-nodeport.yaml
```

Comprobamos el Service:

```bash
kubectl -n my-nginx get svc my-nginx-nodeport
```

Salida orientativa:

```text
NAME                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
my-nginx-nodeport   NodePort   10.96.200.10    <none>        80:30080/TCP   10s
```

Desde la máquina local o desde otra instancia que tenga acceso de red a los nodos, se puede probar:

```bash
curl http://<IP_PUBLICA_O_PRIVADA_DEL_NODO>:30080
```

En AWS, para que funcione desde Internet, el Security Group de los nodos debe permitir entrada TCP al puerto `30080` desde tu IP.

Ejemplo:

```text
Type: Custom TCP
Port: 30080
Source: tu_ip_publica/32
```

> No abras `30080` a `0.0.0.0/0` salvo que sea un laboratorio controlado y temporal.

También se puede probar desde dentro de un nodo del clúster:

```bash
curl http://127.0.0.1:30080
```

Dependiendo de la configuración de kube-proxy y del nodo, puede ser más fiable probar contra la IP real del nodo:

```bash
curl http://$(hostname -I | awk '{print $1}'):30080
```

---

## 8. Observar reglas de kube-proxy

En clústeres kubeadm, kube-proxy suele implementar Services mediante `iptables` o `ipvs`, según la configuración.

Comprobamos kube-proxy:

```bash
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
kubectl -n kube-system get configmap kube-proxy -o yaml | grep -i mode
```

Si kube-proxy usa `iptables`, en un nodo se pueden inspeccionar reglas relacionadas con Services:

```bash
sudo iptables -t nat -L -n | grep -E 'KUBE-SVC|KUBE-NODEPORT|30080' | head -50
```

Si usa `ipvs`, pueden verse con:

```bash
sudo ipvsadm -Ln
```

> `ipvsadm` puede no estar instalado por defecto.

---

## 9. LoadBalancer en kubeadm sobre EC2

Un Service de tipo `LoadBalancer` necesita integración con un proveedor cloud o con una solución como MetalLB. En un clúster kubeadm instalado directamente sobre EC2, normalmente no habrá un controlador que cree automáticamente un balanceador de AWS.

Por eso, si aplicamos el siguiente manifiesto opcional:

```bash
kubectl apply -f modulo6-service-loadbalancer-opcional.yaml
kubectl -n my-nginx get svc my-nginx-loadbalancer
```

Lo esperable en este entorno es ver algo parecido a:

```text
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
my-nginx-loadbalancer   LoadBalancer   10.96.50.100    <pending>     80:xxxxx/TCP   20s
```

Esto **no es un fallo de nginx ni del Service**. Significa que el clúster no tiene un controlador externo capaz de aprovisionar un Load Balancer.

En EKS, en cambio, un Service `LoadBalancer` sí puede crear un balanceador de AWS porque EKS integra los controladores necesarios.

Eliminamos el Service opcional si lo hemos creado:

```bash
kubectl -n my-nginx delete svc my-nginx-loadbalancer
```

---

## 10. Limpieza

Eliminamos los recursos del módulo:

```bash
kubectl delete namespace my-nginx
```

Comprobamos que se han eliminado:

```bash
kubectl get ns my-nginx
```

---

## Resumen

En este módulo hemos visto que:

- Los Pods tienen IPs propias, pero son efímeras.
- Un Service `ClusterIP` proporciona una IP estable dentro del clúster.
- CoreDNS permite acceder a los Services por nombre.
- Las variables de entorno de Service dependen del orden de creación del Pod y del Service.
- `NodePort` permite acceso externo usando la IP de los nodos y un puerto entre `30000` y `32767`.
- `LoadBalancer` requiere integración cloud o una solución adicional; en kubeadm sobre EC2 normalmente quedará en `<pending>`.
