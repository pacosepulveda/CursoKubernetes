# Solución - Ejercicio 01 - Deployment, ReplicaSet, rollout y rollback

## Parte 1 - Crear el Deployment inicial

Aplicamos el Deployment:

```bash
kubectl apply -f yaml/modulo5-ejercicio01-deployment.yaml
```

Salida esperada aproximada:

```text
deployment.apps/tienda-web created
```

Comprobamos el Deployment:

```bash
kubectl get deployment tienda-web
```

Salida esperada aproximada:

```text
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
tienda-web  3/3     3            3           30s
```

Comprobamos los Pods:

```bash
kubectl get pods -l app=tienda-web -o wide
```

Deberían aparecer 3 Pods en estado `Running`.

Comprobamos los objetos relacionados:

```bash
kubectl get deployment,replicaset,pod -l app=tienda-web
```

Salida esperada aproximada:

```text
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/tienda-web   3/3     3            3           1m

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/tienda-web-xxxxxxxxxx   3         3         3       1m

NAME                              READY   STATUS    RESTARTS   AGE
pod/tienda-web-xxxxxxxxxx-aaaaa   1/1     Running   0          1m
pod/tienda-web-xxxxxxxxxx-bbbbb   1/1     Running   0          1m
pod/tienda-web-xxxxxxxxxx-ccccc   1/1     Running   0          1m
```

### Respuestas

1. Hay 3 Pods en ejecución porque el manifiesto define `replicas: 3`.
2. Los Pods los crea y mantiene un `ReplicaSet`.
3. El `Deployment` no gestiona los Pods directamente. Crea un `ReplicaSet`, y el `ReplicaSet` mantiene el número deseado de Pods.

## Parte 2 - Crear el Service

Aplicamos el Service:

```bash
kubectl apply -f yaml/modulo5-ejercicio01-service.yaml
```

Comprobamos el Service:

```bash
kubectl get service tienda-web
```

Salida esperada aproximada:

```text
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
tienda-web   NodePort   10.96.x.y       <none>        80:30081/TCP   10s
```

Probamos el acceso desde dentro del clúster:

```bash
kubectl run curl-tienda --rm -it --restart=Never --image=curlimages/curl:8.8.0 -- \
  curl -I http://tienda-web.default.svc.cluster.local
```

Salida esperada aproximada:

```text
HTTP/1.1 200 OK
Server: nginx/1.25.x
```

El Service proporciona un punto estable de acceso a los Pods seleccionados por la etiqueta `app=tienda-web`.

## Parte 3 - Escalar el Deployment

Escalamos a 5 réplicas:

```bash
kubectl scale deployment/tienda-web --replicas=5
```

Comprobamos los Pods:

```bash
kubectl get pods -l app=tienda-web -o wide
```

Ahora deberían aparecer 5 Pods.

Comprobamos el Deployment:

```bash
kubectl get deployment tienda-web
```

Salida esperada aproximada:

```text
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
tienda-web   5/5     5            5           3m
```

Consultamos el historial:

```bash
kubectl rollout history deployment/tienda-web
```

### Respuestas

1. Escalar el Deployment normalmente no crea una nueva revisión de rollout.
2. El motivo es que el escalado cambia `spec.replicas`, pero no cambia la plantilla del Pod, es decir, no modifica `.spec.template`.

Las revisiones se crean cuando cambia la plantilla del Pod, por ejemplo al cambiar la imagen del contenedor, etiquetas de la plantilla, comandos, variables de entorno, etc.

## Parte 4 - Actualizar correctamente la imagen

Anotamos la causa del cambio:

```bash
kubectl annotate deployment/tienda-web \
  kubernetes.io/change-cause="Actualización correcta a nginx:1.27-alpine" \
  --overwrite
```

Actualizamos la imagen:

```bash
kubectl set image deployment/tienda-web web=nginx:1.27-alpine
```

Observamos el rollout:

```bash
kubectl rollout status deployment/tienda-web
```

Salida esperada aproximada:

```text
deployment "tienda-web" successfully rolled out
```

Comprobamos los ReplicaSets:

```bash
kubectl get replicasets -l app=tienda-web
```

Deberían verse al menos dos ReplicaSets: uno antiguo, normalmente con 0 réplicas, y uno nuevo con las réplicas actuales.

Consultamos el historial:

```bash
kubectl rollout history deployment/tienda-web
```

### Respuestas

1. Sí, se crea un ReplicaSet nuevo porque ha cambiado la plantilla del Pod al cambiar la imagen del contenedor.
2. El Deployment conserva ReplicaSets anteriores para poder realizar rollback.
3. `kubectl rollout history` permite consultar las revisiones disponibles y entender los cambios realizados.

## Parte 5 - Provocar una actualización fallida

Aplicamos el manifiesto con la imagen incorrecta:

```bash
kubectl apply -f yaml/modulo5-ejercicio01-imagen-incorrecta.yaml
```

Comprobamos el estado:

```bash
kubectl get deployment tienda-web
kubectl get pods -l app=tienda-web -o wide
```

Es esperable ver Pods nuevos en estados como:

```text
ImagePullBackOff
ErrImagePull
```

Seleccionamos uno de los Pods afectados y consultamos sus eventos:

```bash
kubectl describe pod NOMBRE_DEL_POD
```

En la parte final, en `Events`, debería aparecer un error indicando que Kubernetes no puede descargar la imagen:

```text
Failed to pull image "nginx:version-inexistente-modulo5"
```

Consultamos el rollout:

```bash
kubectl rollout status deployment/tienda-web
```

Puede quedarse esperando, porque la actualización no puede completarse.

### Respuestas

1. Los Pods nuevos muestran errores de descarga de imagen, como `ImagePullBackOff` o `ErrImagePull`.
2. El evento indica que la imagen no existe o no puede descargarse.
3. Sí, normalmente siguen existiendo Pods disponibles de la versión anterior, porque la estrategia `RollingUpdate` evita eliminar todos los Pods antiguos antes de que los nuevos estén disponibles.
4. La ventaja de `RollingUpdate` es que reduce el impacto de una actualización fallida. Si los Pods nuevos no arrancan, Kubernetes no sustituye toda la aplicación de golpe.

## Parte 6 - Rollback

Revertimos el Deployment:

```bash
kubectl rollout undo deployment/tienda-web
```

Comprobamos el rollout:

```bash
kubectl rollout status deployment/tienda-web
```

Comprobamos los Pods:

```bash
kubectl get pods -l app=tienda-web -o wide
```

Deberían desaparecer los Pods con imagen incorrecta y volver a estar disponibles los Pods con una imagen válida.

Consultamos el historial:

```bash
kubectl rollout history deployment/tienda-web
```

### Respuestas

1. Los Pods que usaban la imagen incorrecta se eliminan porque el Deployment vuelve a la plantilla anterior.
2. Sí, el Deployment vuelve a estar disponible cuando las réplicas correctas están en estado `Running` y `Ready`.
3. El historial de revisiones es útil porque permite volver rápidamente a una versión anterior conocida y funcional.

## Limpieza

Eliminamos el Service y el Deployment:

```bash
kubectl delete -f yaml/modulo5-ejercicio01-service.yaml --ignore-not-found
kubectl delete -f yaml/modulo5-ejercicio01-deployment.yaml --ignore-not-found
```

Comprobamos:

```bash
kubectl get deployment,replicaset,pod,service -l app=tienda-web
```

No deberían aparecer recursos asociados a `app=tienda-web`.

> Si quedara algún Pod temporal de curl, se puede eliminar con:
>
> ```bash
> kubectl delete pod curl-tienda --ignore-not-found
> ```
