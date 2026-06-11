# Ejercicios Módulo 12 - Ingress

## Objetivo

En este módulo vamos a instalar y probar un **Ingress Controller NGINX** en un clúster Kubernetes instalado con `kubeadm` sobre instancias EC2.

El objetivo es entender la diferencia entre:

- `Service` de tipo `ClusterIP`: expone una aplicación solo dentro del clúster.
- `Service` de tipo `NodePort`: expone una aplicación en un puerto de los nodos.
- `Ingress`: permite enrutar tráfico HTTP/HTTPS hacia distintos servicios usando reglas de host y/o path.

> En un clúster EKS es habitual exponer el Ingress Controller con un `LoadBalancer` de AWS. En este laboratorio, al usar `kubeadm` sobre EC2, normalmente no hay integración automática con AWS Load Balancer, por lo que instalaremos `ingress-nginx` con `Service` de tipo `NodePort`.

---

## 1. Comprobar requisitos previos

Ejecuta desde el nodo desde el que tengas configurado `kubectl`:

```bash
kubectl get nodes -o wide
kubectl get pods -A
helm version --short
```

También comprobaremos que no hay un Ingress Controller instalado previamente:

```bash
kubectl get ns ingress-nginx
kubectl get ingressclass
```

Si el namespace no existe, es normal.

---

## 2. Instalar ingress-nginx con Helm

Añadimos el repositorio oficial del proyecto `ingress-nginx`:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Instalamos el controlador usando `NodePort`:

```bash
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443
```

Esperamos a que el controlador esté disponible:

```bash
kubectl -n ingress-nginx rollout status deployment/ingress-nginx-controller
```

Comprobamos los recursos creados:

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
kubectl get ingressclass
```

La salida del servicio debe mostrar algo parecido a esto:

```text
NAME                                 TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)
ingress-nginx-controller             NodePort   10.x.x.x        <none>        80:30080/TCP,443:30443/TCP
```

> Si aparecen `LoadBalancer` y `EXTERNAL-IP: <pending>`, la instalación no se ha adaptado correctamente al entorno kubeadm. Reinstala el chart usando los parámetros anteriores.

---

## 3. Preparar el namespace del ejercicio

Crearemos todos los recursos de aplicación en un namespace separado:

```bash
kubectl create namespace ingress-demo
```

---

## 4. Desplegar la aplicación 2048

Aplicamos el manifiesto local:

```bash
kubectl apply -f modulo12-2048.yaml
```

Comprobamos que el Deployment, el Pod y el Service están funcionando:

```bash
kubectl -n ingress-demo get deploy,pod,svc -o wide
```

El servicio `service-2048` es de tipo `ClusterIP`, porque no queremos exponer directamente la aplicación. La expondremos a través del Ingress Controller.

Prueba interna desde un pod temporal:

```bash
kubectl -n ingress-demo run curl-test --rm -it --restart=Never \
  --image=curlimages/curl:8.10.1 -- \
  curl -I http://service-2048
```

Debe devolver una respuesta HTTP, normalmente `HTTP/1.1 200 OK`.

---

## 5. Desplegar la aplicación hello

Aplicamos el segundo manifiesto:

```bash
kubectl apply -f modulo12-hello.yaml
```

Comprobamos los recursos:

```bash
kubectl -n ingress-demo get deploy,pod,svc -o wide
```

Prueba interna:

```bash
kubectl -n ingress-demo run curl-test --rm -it --restart=Never \
  --image=curlimages/curl:8.10.1 -- \
  curl http://hello-world
```

---

## 6. Crear el primer Ingress: raíz hacia 2048

Aplicamos el primer Ingress:

```bash
kubectl apply -f modulo12-ingress-2048.yaml
```

Comprobamos que se ha creado:

```bash
kubectl -n ingress-demo get ingress
kubectl -n ingress-demo describe ingress demo-ingress-2048
```

Como no estamos usando un DNS real, accederemos usando la IP pública de cualquier nodo y el puerto NodePort `30080`.

Primero, localiza una IP pública de un nodo EC2:

```bash
kubectl get nodes -o wide
```

Guarda la IP pública de uno de los nodos en una variable:

```bash
export NODE_PUBLIC_IP=<IP_PUBLICA_DE_UN_NODO>
```

Prueba con `curl`:

```bash
curl -H 'Host: demo.local' http://$NODE_PUBLIC_IP:30080/
```

También puedes probarlo desde el navegador:

```text
http://<IP_PUBLICA_DE_UN_NODO>:30080/
```

En el navegador puede funcionar sin cabecera `Host` porque el Ingress también incluye una regla sin host. Con `curl` usamos `Host: demo.local` para mostrar explícitamente cómo funciona el enrutamiento por host.

---

## 7. Añadir enrutamiento por path: `/hello`

Ahora vamos a actualizar el mismo Ingress creado en el paso anterior.

En el paso 6 creamos un Ingress llamado `demo-ingress-2048` que enviaba la ruta `/` hacia el servicio `service-2048`.

Ahora vamos a aplicar una versión ampliada de ese mismo Ingress para que:

- `/` apunte al servicio `service-2048`.
- `/hello` apunte al servicio `hello-world`.

> Importante: el manifiesto `modulo12-ingress-paths.yaml` debe usar el mismo nombre de Ingress que el manifiesto anterior:
>
> `metadata.name: demo-ingress-2048`
>
> Si se usara otro nombre de Ingress con el mismo `host` y el mismo `path`, el webhook de validación de `ingress-nginx` rechazaría el manifiesto porque ya existiría una regla para `demo.local` y `/`.

Antes de aplicar el cambio, podemos ver el Ingress actual:

```bash
kubectl -n ingress-demo get ingress
kubectl -n ingress-demo describe ingress demo-ingress-2048
```

Observa que actualmente solo existe la ruta `/`.

Aplicamos el manifiesto completo:

```bash
kubectl apply -f modulo12-ingress-paths.yaml
```

Comprobamos el Ingress:

```bash
kubectl -n ingress-demo describe ingress demo-ingress-2048
```

Probamos la ruta raíz:

```bash
curl -I -H 'Host: demo.local' http://$NODE_PUBLIC_IP:30080/
```

Probamos la ruta `/hello`:

```bash
curl -H 'Host: demo.local' http://$NODE_PUBLIC_IP:30080/hello
```

La petición a `/hello` debe responder desde la aplicación `hello-world`.

---

## 8. Sobre `rewrite-target`

Muchas aplicaciones esperan recibir las peticiones en `/`. Si publicamos una aplicación bajo `/hello`, el backend recibirá por defecto la ruta `/hello`.

Para aplicaciones que no soportan ese prefijo, podemos usar la anotación de NGINX:

```yaml
nginx.ingress.kubernetes.io/rewrite-target: /
```

En este módulo usamos esa anotación en `modulo12-ingress-paths.yaml` para que `/hello` se reescriba como `/` antes de llegar al servicio `hello-world`.

> Importante: `rewrite-target` se aplica al Ingress completo. En escenarios reales, cuando distintas rutas necesitan comportamientos distintos, suele ser más limpio crear Ingress separados.

---

## 9. Ejercicio: crear un Ingress separado para hello

Crea un nuevo Ingress llamado `hello-ingress` que publique `hello-world` bajo el host `hello.local`.

Debe cumplir estas condiciones:

- Namespace: `ingress-demo`
- `ingressClassName: nginx`
- Host: `hello.local`
- Path: `/`
- Servicio backend: `hello-world`
- Puerto backend: `80`

Después pruébalo con:

```bash
curl -H 'Host: hello.local' http://$NODE_PUBLIC_IP:30080/
```

Solución propuesta:

```bash
kubectl apply -f modulo12-ingress-hello-host.yaml
curl -H 'Host: hello.local' http://$NODE_PUBLIC_IP:30080/
```

---

## 10. Diagnóstico básico de Ingress

Si no funciona, revisa en este orden:

```bash
# El controlador está Running
kubectl -n ingress-nginx get pods

# El Service del controlador expone el puerto 30080
kubectl -n ingress-nginx get svc ingress-nginx-controller

# Existe una IngressClass nginx
kubectl get ingressclass

# Los backends existen y tienen endpoints
kubectl -n ingress-demo get svc
kubectl -n ingress-demo get endpoints

# El Ingress tiene reglas correctas
kubectl -n ingress-demo describe ingress

# Logs del controlador
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller --tail=100
```

En AWS EC2, recuerda comprobar también el Security Group de los nodos. Para acceder desde tu equipo, el Security Group debe permitir tráfico entrante al puerto `30080/TCP` desde tu IP.

---

## 11. Limpieza

Eliminamos los recursos del ejercicio:

```bash
kubectl delete namespace ingress-demo
```

Si también quieres eliminar el Ingress Controller:

```bash
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace ingress-nginx
```

---

## Resumen

En este módulo hemos visto:

- Qué problema resuelve Ingress.
- Cómo instalar `ingress-nginx` con Helm.
- Por qué en kubeadm sobre EC2 usamos `NodePort` en lugar de `LoadBalancer`.
- Cómo enrutar tráfico HTTP por path.
- Cómo diagnosticar problemas habituales de Ingress.
