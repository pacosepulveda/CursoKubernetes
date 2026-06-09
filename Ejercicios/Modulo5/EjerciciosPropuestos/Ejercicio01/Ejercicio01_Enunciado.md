# Ejercicio 01 - Deployment, ReplicaSet, rollout y rollback

## Objetivo

Practicar el ciclo de vida básico de un `Deployment`:

- Crear un `Deployment` de forma declarativa.
- Comprobar los `ReplicaSet` generados por el `Deployment`.
- Escalar manualmente el número de réplicas.
- Realizar una actualización de imagen.
- Provocar un fallo de actualización.
- Diagnosticar el problema con `kubectl get`, `describe` y `rollout`.
- Revertir el `Deployment` a una revisión anterior.

## Archivos proporcionados

En la carpeta `yaml/` tienes estos archivos:

```text
modulo5-ejercicio01-deployment.yaml
modulo5-ejercicio01-service.yaml
modulo5-ejercicio01-imagen-incorrecta.yaml
```

## Contexto

La aplicación `tienda-web` representa un frontend web sin estado. Se desplegará con 3 réplicas mediante un `Deployment`.

El `Deployment` usa una estrategia `RollingUpdate` con:

- `maxUnavailable: 1`
- `maxSurge: 1`

Esto significa que durante una actualización Kubernetes puede crear como máximo un Pod adicional y puede permitir como máximo un Pod no disponible.

## Parte 1 - Crear el Deployment inicial

Aplica el Deployment:

```bash
kubectl apply -f yaml/modulo5-ejercicio01-deployment.yaml
```

Comprueba el estado del Deployment:

```bash
kubectl get deployment tienda-web
```

Comprueba los Pods:

```bash
kubectl get pods -l app=tienda-web -o wide
```

Comprueba los objetos relacionados:

```bash
kubectl get deployment,replicaset,pod -l app=tienda-web
```

### Preguntas

1. ¿Cuántos Pods hay en ejecución?
2. ¿Qué objeto ha creado los Pods realmente?
3. ¿El Deployment crea directamente los Pods o lo hace mediante un ReplicaSet?

## Parte 2 - Crear el Service

Aplica el Service:

```bash
kubectl apply -f yaml/modulo5-ejercicio01-service.yaml
```

Comprueba el Service:

```bash
kubectl get service tienda-web
```

Prueba el acceso desde dentro del clúster:

```bash
kubectl run curl-tienda --rm -it --restart=Never --image=curlimages/curl:8.8.0 -- \
  curl -I http://tienda-web.default.svc.cluster.local
```

## Parte 3 - Escalar el Deployment

Escala el Deployment a 5 réplicas:

```bash
kubectl scale deployment/tienda-web --replicas=5
```

Comprueba los Pods:

```bash
kubectl get pods -l app=tienda-web -o wide
```

Comprueba el Deployment:

```bash
kubectl get deployment tienda-web
```

Consulta el historial de rollout:

```bash
kubectl rollout history deployment/tienda-web
```

### Preguntas

1. ¿Escalar el Deployment ha creado una nueva revisión?
2. ¿Por qué?

## Parte 4 - Actualizar correctamente la imagen

Anota la causa del cambio:

```bash
kubectl annotate deployment/tienda-web \
  kubernetes.io/change-cause="Actualización correcta a nginx:1.27-alpine" \
  --overwrite
```

Actualiza la imagen del contenedor:

```bash
kubectl set image deployment/tienda-web web=nginx:1.27-alpine
```

Observa el rollout:

```bash
kubectl rollout status deployment/tienda-web
```

Comprueba los ReplicaSets:

```bash
kubectl get replicasets -l app=tienda-web
```

Consulta el historial:

```bash
kubectl rollout history deployment/tienda-web
```

### Preguntas

1. ¿Se ha creado un ReplicaSet nuevo?
2. ¿Por qué el Deployment conserva ReplicaSets anteriores?
3. ¿Para qué sirve `kubectl rollout history`?

## Parte 5 - Provocar una actualización fallida

Aplica el manifiesto con una imagen incorrecta:

```bash
kubectl apply -f yaml/modulo5-ejercicio01-imagen-incorrecta.yaml
```

Espera unos segundos y comprueba el estado:

```bash
kubectl get deployment tienda-web
kubectl get pods -l app=tienda-web -o wide
```

Consulta los eventos de un Pod afectado:

```bash
kubectl describe pod NOMBRE_DEL_POD
```

Consulta el estado del rollout:

```bash
kubectl rollout status deployment/tienda-web
```

> Puedes detener el comando con `Ctrl+C` si se queda esperando demasiado tiempo.

### Preguntas

1. ¿Qué estado aparece en los Pods nuevos?
2. ¿Qué indica el evento relacionado con la imagen?
3. ¿Sigue habiendo Pods disponibles de la versión anterior?
4. ¿Qué ventaja aporta la estrategia `RollingUpdate` en este fallo?

## Parte 6 - Rollback

Revierte el Deployment a la revisión anterior:

```bash
kubectl rollout undo deployment/tienda-web
```

Comprueba el estado del rollout:

```bash
kubectl rollout status deployment/tienda-web
```

Comprueba los Pods:

```bash
kubectl get pods -l app=tienda-web -o wide
```

Consulta el historial:

```bash
kubectl rollout history deployment/tienda-web
```

### Preguntas

1. ¿Qué ha ocurrido con los Pods que usaban la imagen incorrecta?
2. ¿El Deployment vuelve a estar disponible?
3. ¿Por qué es útil mantener el historial de revisiones?

## Limpieza

Elimina los recursos creados:

```bash
kubectl delete -f yaml/modulo5-ejercicio01-service.yaml --ignore-not-found
kubectl delete -f yaml/modulo5-ejercicio01-deployment.yaml --ignore-not-found
```

Comprueba que no quedan recursos principales del ejercicio:

```bash
kubectl get deployment,replicaset,pod,service -l app=tienda-web
```
