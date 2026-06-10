# Ejercicio 01 - Troubleshooting de Services, DNS y NodePort en Kubernetes

## Contexto del ejercicio

El equipo de desarrollo ha desplegado una pequeña aplicación web llamada **catalog-web** en un clúster Kubernetes.

La aplicación debería cumplir estos requisitos:

- Ejecutarse en el namespace `svc-troubleshooting`.
- Tener 2 réplicas de un servidor web `nginx`.
- Ser accesible desde dentro del namespace mediante el nombre DNS:

```bash
http://catalog-web
```

- Ser accesible desde otro namespace usando el FQDN completo del Service:

```bash
http://catalog-web.svc-troubleshooting.svc.cluster.local
```

- Ser accesible desde fuera del clúster mediante un Service de tipo `NodePort` en el puerto `30081`.

Sin embargo, el despliegue actual **no funciona correctamente**. Tu tarea consiste en diagnosticar los problemas y corregirlos.

> Este ejercicio está pensado para practicar troubleshooting. No modifiques los YAML al principio. Primero observa, recopila evidencias y explica qué está ocurriendo.

---

## Objetivos

Al finalizar el ejercicio deberías ser capaz de:

- Diagnosticar por qué un Service no tiene endpoints.
- Relacionar los selectores de un Service con las etiquetas de los Pods.
- Comprobar `EndpointSlice` y entender a qué Pods y puertos apunta un Service.
- Detectar un `targetPort` incorrecto.
- Diferenciar resolución DNS dentro del mismo namespace y entre namespaces distintos.
- Diagnosticar un `NodePort` que no enruta correctamente hacia los Pods.
- Corregir los errores aplicando manifiestos YAML ya preparados.

---

## Estructura de archivos

```text
Ejercicio01/
├── Ejercicio01_Enunciado.md
├── Ejercicio01_Solucion.md
└── yaml/
    ├── 00-namespace.yaml
    ├── 01-catalog-web-deployment.yaml
    ├── 02-catalog-web-service-broken.yaml
    ├── 03-client-pods.yaml
    ├── 04-catalog-web-nodeport-broken.yaml
    ├── 90-cleanup.yaml
    └── fix/
        ├── 01-catalog-web-service-fixed.yaml
        └── 02-catalog-web-nodeport-fixed.yaml
```

---

## 1. Desplegar el escenario con errores

Sitúate en la carpeta del ejercicio y aplica los manifiestos iniciales:

```bash
kubectl apply -f yaml/00-namespace.yaml
kubectl apply -f yaml/01-catalog-web-deployment.yaml
kubectl apply -f yaml/02-catalog-web-service-broken.yaml
kubectl apply -f yaml/03-client-pods.yaml
kubectl apply -f yaml/04-catalog-web-nodeport-broken.yaml
```

Espera a que los Pods estén en estado `Running`:

```bash
kubectl -n svc-troubleshooting get pods -o wide
kubectl -n testing-tools get pods -o wide
```

---

## 2. Comprobaciones iniciales

Comprueba el Deployment:

```bash
kubectl -n svc-troubleshooting get deployment catalog-web
kubectl -n svc-troubleshooting get pods --show-labels
```

Comprueba los Services:

```bash
kubectl -n svc-troubleshooting get svc
kubectl -n svc-troubleshooting describe svc catalog-web
kubectl -n svc-troubleshooting describe svc catalog-web-nodeport
```

Comprueba si los Services tienen endpoints:

```bash
kubectl -n svc-troubleshooting get endpoints
kubectl -n svc-troubleshooting get endpointslices
```

---

## 3. Prueba de acceso desde el mismo namespace

Desde el Pod cliente del mismo namespace, prueba el acceso al Service `catalog-web`:

```bash
kubectl -n svc-troubleshooting exec client -- curl -sS -I --connect-timeout 3 http://catalog-web
```

La petición no debería funcionar correctamente.

Investiga:

```bash
kubectl -n svc-troubleshooting get svc catalog-web -o yaml
kubectl -n svc-troubleshooting get pods --show-labels
kubectl -n svc-troubleshooting get endpointslices -l kubernetes.io/service-name=catalog-web -o yaml
```

Preguntas:

1. ¿El Service `catalog-web` está seleccionando algún Pod?
2. ¿Qué etiquetas tienen realmente los Pods?
3. ¿Qué selector está usando el Service?
4. ¿Qué habría que corregir?

---

## 4. Corregir el primer problema

Aplica el manifiesto corregido del Service `ClusterIP`:

```bash
kubectl apply -f yaml/fix/01-catalog-web-service-fixed.yaml
```

Vuelve a comprobar:

```bash
kubectl -n svc-troubleshooting describe svc catalog-web
kubectl -n svc-troubleshooting get endpointslices -l kubernetes.io/service-name=catalog-web -o wide
```

Prueba de nuevo:

```bash
kubectl -n svc-troubleshooting exec client -- curl -sS -I --connect-timeout 3 http://catalog-web
```

La petición debería devolver una respuesta HTTP de nginx.

---

## 5. Prueba DNS desde otro namespace

Hay otro Pod cliente en el namespace `testing-tools`.

Prueba primero con el nombre corto del Service:

```bash
kubectl -n testing-tools exec external-client -- curl -sS -I --connect-timeout 3 http://catalog-web
```

Esta prueba debería fallar.

Ahora prueba usando el FQDN completo:

```bash
kubectl -n testing-tools exec external-client -- curl -sS -I --connect-timeout 3 http://catalog-web.svc-troubleshooting.svc.cluster.local
```

Preguntas:

1. ¿Por qué `catalog-web` funciona desde el mismo namespace pero no desde `testing-tools`?
2. ¿Qué nombre DNS debe usarse entre namespaces?
3. ¿Qué patrón sigue el FQDN de un Service de Kubernetes?

---

## 6. Troubleshooting del NodePort

El Service `catalog-web-nodeport` debería publicar la aplicación en el puerto `30081` de los nodos.

Comprueba el Service:

```bash
kubectl -n svc-troubleshooting get svc catalog-web-nodeport
kubectl -n svc-troubleshooting describe svc catalog-web-nodeport
kubectl -n svc-troubleshooting get endpointslices -l kubernetes.io/service-name=catalog-web-nodeport -o yaml
```

Desde un nodo del clúster, prueba:

```bash
curl -sS -I --connect-timeout 3 http://127.0.0.1:30081
```

Si no funciona con `127.0.0.1`, prueba con la IP real del nodo:

```bash
curl -sS -I --connect-timeout 3 http://$(hostname -I | awk '{print $1}'):30081
```

Preguntas:

1. ¿El Service `NodePort` selecciona Pods?
2. ¿A qué puerto de los Pods está enviando tráfico?
3. ¿En qué puerto escucha realmente nginx dentro del contenedor?
4. ¿Qué habría que corregir?

---

## 7. Corregir el NodePort

Aplica el manifiesto corregido:

```bash
kubectl apply -f yaml/fix/02-catalog-web-nodeport-fixed.yaml
```

Comprueba:

```bash
kubectl -n svc-troubleshooting describe svc catalog-web-nodeport
kubectl -n svc-troubleshooting get endpointslices -l kubernetes.io/service-name=catalog-web-nodeport -o wide
```

Prueba de nuevo desde un nodo:

```bash
curl -sS -I --connect-timeout 3 http://$(hostname -I | awk '{print $1}'):30081
```

Salida esperada orientativa:

```text
HTTP/1.1 200 OK
Server: nginx/...
```

> Si estás en AWS y quieres acceder desde Internet usando la IP pública del nodo, recuerda que el Security Group debe permitir entrada TCP al puerto `30081` desde tu IP pública. No abras el puerto a `0.0.0.0/0` salvo que sea un laboratorio controlado y temporal.

---

## 8. Limpieza

Cuando termines el ejercicio:

```bash
kubectl apply -f yaml/90-cleanup.yaml
```

O bien:

```bash
kubectl delete namespace svc-troubleshooting
kubectl delete namespace testing-tools
```

---
