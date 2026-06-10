# Ejercicio 01 - Solución explicada

## 1. Despliegue inicial

Aplicamos los manifiestos con errores:

```bash
kubectl apply -f yaml/00-namespace.yaml
kubectl apply -f yaml/01-catalog-web-deployment.yaml
kubectl apply -f yaml/02-catalog-web-service-broken.yaml
kubectl apply -f yaml/03-client-pods.yaml
kubectl apply -f yaml/04-catalog-web-nodeport-broken.yaml
```

Comprobamos los Pods:

```bash
kubectl -n svc-troubleshooting get pods -o wide
kubectl -n testing-tools get pods -o wide
```

Deberíamos ver algo parecido a:

```text
NAME                           READY   STATUS    RESTARTS   AGE   IP            NODE
catalog-web-xxxxxxxxxx-aaaaa   1/1     Running   0          30s   10.244.1.10   worker1
catalog-web-xxxxxxxxxx-bbbbb   1/1     Running   0          30s   10.244.2.12   worker2
client                         1/1     Running   0          30s   10.244.1.20   worker1
```

El hecho de que los Pods estén en `Running` no significa que la aplicación esté correctamente publicada mediante Services.

---

## 2. Problema 1: el Service `catalog-web` no tiene endpoints

Probamos el acceso desde el Pod cliente del mismo namespace:

```bash
kubectl -n svc-troubleshooting exec client -- curl -sS -I --connect-timeout 3 http://catalog-web
```

El acceso falla.

Lo primero es comprobar el Service:

```bash
kubectl -n svc-troubleshooting describe svc catalog-web
```

En la salida veremos algo importante:

```text
Selector:          app=catalog
Endpoints:         <none>
```

Esto indica que el Service existe, tiene una `ClusterIP`, tiene un puerto configurado, pero **no está apuntando a ningún Pod**.

Comprobamos las etiquetas reales de los Pods:

```bash
kubectl -n svc-troubleshooting get pods --show-labels
```

Salida orientativa:

```text
NAME                           READY   STATUS    LABELS
catalog-web-xxxxxxxxxx-aaaaa   1/1     Running   app=catalog-web,component=frontend,pod-template-hash=...
catalog-web-xxxxxxxxxx-bbbbb   1/1     Running   app=catalog-web,component=frontend,pod-template-hash=...
```

Los Pods tienen la etiqueta:

```text
app=catalog-web
```

Pero el Service está buscando:

```text
app=catalog
```

Por eso el Service no tiene endpoints.

También podemos verlo con `EndpointSlice`:

```bash
kubectl -n svc-troubleshooting get endpointslices -l kubernetes.io/service-name=catalog-web -o yaml
```

Si el selector no coincide con ningún Pod, no habrá direcciones útiles asociadas al Service.

### Corrección

Aplicamos el Service corregido:

```bash
kubectl apply -f yaml/fix/01-catalog-web-service-fixed.yaml
```

El cambio principal es este:

```yaml
selector:
  app: catalog-web
  component: frontend
```

Además, el `targetPort` se corrige para apuntar al puerto real del contenedor:

```yaml
targetPort: 80
```

Comprobamos otra vez:

```bash
kubectl -n svc-troubleshooting describe svc catalog-web
kubectl -n svc-troubleshooting get endpointslices -l kubernetes.io/service-name=catalog-web -o wide
```

Ahora el Service debería mostrar endpoints.

Probamos el acceso interno:

```bash
kubectl -n svc-troubleshooting exec client -- curl -sS -I --connect-timeout 3 http://catalog-web
```

Salida esperada:

```text
HTTP/1.1 200 OK
Server: nginx/...
```

---

## 3. Problema 2: nombre DNS corto desde otro namespace

Desde el namespace `testing-tools` probamos:

```bash
kubectl -n testing-tools exec external-client -- curl -sS -I --connect-timeout 3 http://catalog-web
```

Esta prueba falla porque el Pod `external-client` está en otro namespace.

Cuando un Pod resuelve un nombre corto como:

```text
catalog-web
```

Kubernetes intenta resolverlo primero dentro de su propio namespace. En este caso, el cliente está en `testing-tools`, así que intentará encontrar un Service llamado:

```text
catalog-web.testing-tools.svc.cluster.local
```

Ese Service no existe.

Para acceder a un Service que está en otro namespace hay que usar el nombre cualificado:

```bash
kubectl -n testing-tools exec external-client -- curl -sS -I --connect-timeout 3 http://catalog-web.svc-troubleshooting.svc.cluster.local
```

Salida esperada:

```text
HTTP/1.1 200 OK
Server: nginx/...
```

El patrón general del FQDN de un Service es:

```text
<service>.<namespace>.svc.cluster.local
```

También suele funcionar el nombre parcialmente cualificado:

```text
catalog-web.svc-troubleshooting
```

Pero para un ejercicio didáctico es mejor usar el FQDN completo.

---

## 4. Problema 3: el NodePort apunta al puerto incorrecto

El Service `catalog-web-nodeport` existe y publica el puerto `30081`:

```bash
kubectl -n svc-troubleshooting get svc catalog-web-nodeport
```

Salida orientativa:

```text
NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)
catalog-web-nodeport   NodePort   10.96.x.y       <none>        80:30081/TCP
```

Probamos desde un nodo:

```bash
curl -sS -I --connect-timeout 3 http://$(hostname -I | awk '{print $1}'):30081
```

La prueba falla.

Comprobamos el Service:

```bash
kubectl -n svc-troubleshooting describe svc catalog-web-nodeport
```

La salida muestra que el Service sí selecciona Pods, pero envía el tráfico a un puerto incorrecto:

```text
Port:        http  80/TCP
TargetPort: 8080/TCP
NodePort:    http  30081/TCP
```

El problema es que la imagen `nginx` escucha en el puerto `80`, no en el puerto `8080`.

Esto también puede verse revisando el Deployment:

```bash
kubectl -n svc-troubleshooting get deployment catalog-web -o yaml | grep -A5 ports
```

O describiendo uno de los Pods:

```bash
kubectl -n svc-troubleshooting describe pod -l app=catalog-web
```

El contenedor declara:

```yaml
containerPort: 80
```

### Corrección

Aplicamos el manifiesto corregido:

```bash
kubectl apply -f yaml/fix/02-catalog-web-nodeport-fixed.yaml
```

La corrección principal es:

```yaml
targetPort: 80
```

Comprobamos:

```bash
kubectl -n svc-troubleshooting describe svc catalog-web-nodeport
kubectl -n svc-troubleshooting get endpointslices -l kubernetes.io/service-name=catalog-web-nodeport -o wide
```

Probamos de nuevo desde un nodo:

```bash
curl -sS -I --connect-timeout 3 http://$(hostname -I | awk '{print $1}'):30081
```

Salida esperada:

```text
HTTP/1.1 200 OK
Server: nginx/...
```

---

## 5. Resumen de errores encontrados

| Recurso | Error | Síntoma | Corrección |
|---|---|---|---|
| `Service/catalog-web` | Selector `app: catalog` | Service sin endpoints | Usar `app: catalog-web` y `component: frontend` |
| `Service/catalog-web` | `targetPort: 8080` | Aunque seleccionara Pods, enviaría tráfico al puerto equivocado | Usar `targetPort: 80` |
| DNS desde `testing-tools` | Uso del nombre corto `catalog-web` | El cliente busca el Service en su propio namespace | Usar `catalog-web.svc-troubleshooting.svc.cluster.local` |
| `Service/catalog-web-nodeport` | `targetPort: 8080` | NodePort abierto pero sin respuesta válida de nginx | Usar `targetPort: 80` |

---

## 6. Comandos clave de troubleshooting

### Ver Pods y etiquetas

```bash
kubectl -n svc-troubleshooting get pods --show-labels
```

### Describir un Service

```bash
kubectl -n svc-troubleshooting describe svc catalog-web
```

### Ver endpoints clásicos

```bash
kubectl -n svc-troubleshooting get endpoints
```

### Ver EndpointSlices

```bash
kubectl -n svc-troubleshooting get endpointslices
kubectl -n svc-troubleshooting get endpointslices -l kubernetes.io/service-name=catalog-web -o yaml
```

### Probar desde un Pod cliente

```bash
kubectl -n svc-troubleshooting exec client -- curl -sS -I http://catalog-web
```

### Probar desde otro namespace

```bash
kubectl -n testing-tools exec external-client -- curl -sS -I http://catalog-web.svc-troubleshooting.svc.cluster.local
```

### Probar NodePort desde un nodo

```bash
curl -sS -I http://$(hostname -I | awk '{print $1}'):30081
```

---

## 7. Limpieza

```bash
kubectl apply -f yaml/90-cleanup.yaml
```

O bien:

```bash
kubectl delete namespace svc-troubleshooting
kubectl delete namespace testing-tools
```
