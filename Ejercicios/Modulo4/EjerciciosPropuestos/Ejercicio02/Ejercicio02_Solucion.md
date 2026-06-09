# Solución - Ejercicio 2 - Namespaces, Labels y Annotations

## 1. Crear los Namespaces y desplegar la aplicación

Comprobamos primero que tenemos acceso al clúster:

```bash
kubectl get nodes
```

Creamos los Namespaces:

```bash
kubectl create namespace m4-dev
kubectl create namespace m4-prod
```

Aplicamos el mismo archivo YAML en ambos Namespaces:

```bash
kubectl apply -n m4-dev -f yaml/modulo4-ejercicio02-aplicacion.yaml
kubectl apply -n m4-prod -f yaml/modulo4-ejercicio02-aplicacion.yaml
```

El manifiesto es neutral respecto al Namespace. No tiene `metadata.namespace`. Esto permite reutilizar el mismo archivo en distintos entornos.

## 2. Comprobar los Namespaces y los Deployments

Listamos los Namespaces:

```bash
kubectl get namespaces
```

Salida esperada parcial:

```text
NAME              STATUS   AGE
default           Active   ...
kube-node-lease   Active   ...
kube-public       Active   ...
kube-system       Active   ...
m4-dev            Active   ...
m4-prod           Active   ...
```

Comprobamos los Deployments:

```bash
kubectl get deployments -n m4-dev
kubectl get deployments -n m4-prod
```

Salida esperada aproximada en ambos Namespaces:

```text
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
portal-backend    1/1     1            1           30s
portal-frontend   2/2     2            2           30s
```

Los Deployments tienen el mismo nombre en `m4-dev` y en `m4-prod`. Kubernetes lo permite porque los nombres de los recursos namespaced solo deben ser únicos dentro de su propio Namespace.

## 3. Consultar Pods y Labels

Ejecutamos:

```bash
kubectl get pods -n m4-dev --show-labels
kubectl get pods -n m4-prod --show-labels
```

Salida esperada aproximada:

```text
NAME                               READY   STATUS    RESTARTS   AGE   LABELS
portal-backend-xxxxxxxxxx-yyyyy    1/1     Running   0          1m    app=portal,pod-template-hash=...,tier=backend
portal-frontend-xxxxxxxxxx-aaaaa   1/1     Running   0          1m    app=portal,pod-template-hash=...,tier=frontend
portal-frontend-xxxxxxxxxx-bbbbb   1/1     Running   0          1m    app=portal,pod-template-hash=...,tier=frontend
```

Interpretación:

- La Label `app=portal` identifica todos los Pods de la aplicación.
- La Label `tier=frontend` identifica los Pods del frontend.
- La Label `tier=backend` identifica los Pods del backend.
- La Label `pod-template-hash` la añade Kubernetes para relacionar los Pods con el ReplicaSet correspondiente.

## 4. Filtrar recursos por Labels

Frontend en desarrollo:

```bash
kubectl get pods -n m4-dev -l app=portal,tier=frontend
```

Salida esperada aproximada:

```text
NAME                               READY   STATUS    RESTARTS   AGE
portal-frontend-xxxxxxxxxx-aaaaa   1/1     Running   0          2m
portal-frontend-xxxxxxxxxx-bbbbb   1/1     Running   0          2m
```

Backend en producción:

```bash
kubectl get pods -n m4-prod -l app=portal,tier=backend
```

Salida esperada aproximada:

```text
NAME                              READY   STATUS    RESTARTS   AGE
portal-backend-xxxxxxxxxx-yyyyy   1/1     Running   0          2m
```

Todos los Pods de la aplicación en todos los Namespaces:

```bash
kubectl get pods -A -l app=portal
```

Deberíamos ver seis Pods en total:

- Tres en `m4-dev`.
- Tres en `m4-prod`.

Las Labels permiten seleccionar grupos lógicos de recursos sin depender de nombres concretos de Pods, que en el caso de Deployments son dinámicos y cambian si se recrean.

## 5. Añadir Labels manualmente a Deployments

Etiquetamos los Deployments de desarrollo:

```bash
kubectl label deployment -n m4-dev -l app=portal entorno=desarrollo
```

Etiquetamos los Deployments de producción:

```bash
kubectl label deployment -n m4-prod -l app=portal entorno=produccion
```

Comprobamos:

```bash
kubectl get deployments -n m4-dev --show-labels
kubectl get deployments -n m4-prod --show-labels
```

Salida esperada aproximada en `m4-dev`:

```text
NAME              READY   UP-TO-DATE   AVAILABLE   AGE   LABELS
portal-backend    1/1     1            1           3m    app=portal,entorno=desarrollo,tier=backend
portal-frontend   2/2     2            2           3m    app=portal,entorno=desarrollo,tier=frontend
```

Salida esperada aproximada en `m4-prod`:

```text
NAME              READY   UP-TO-DATE   AVAILABLE   AGE   LABELS
portal-backend    1/1     1            1           3m    app=portal,entorno=produccion,tier=backend
portal-frontend   2/2     2            2           3m    app=portal,entorno=produccion,tier=frontend
```

La Label se ha aplicado a los objetos Deployment. Eso no significa necesariamente que se haya aplicado a los Pods existentes.

Para comprobarlo:

```bash
kubectl get pods -n m4-dev --show-labels
kubectl get pods -n m4-prod --show-labels
```

Los Pods seguirán teniendo las Labels definidas en la plantilla del Deployment, principalmente `app=portal` y `tier=frontend/backend`.

Idea clave:

- Las Labels de `metadata.labels` del Deployment describen el objeto Deployment.
- Las Labels de `spec.template.metadata.labels` son las que se copian a los Pods creados por el Deployment.

## 6. Añadir Annotations descriptivas

Añadimos una Annotation a los Deployments de desarrollo:

```bash
kubectl annotate deployment -n m4-dev -l app=portal curso.kubernetes.io/nota="Entorno usado para practicar Namespaces, Labels y Annotations"
```

Consultamos el Deployment:

```bash
kubectl describe deployment portal-frontend -n m4-dev
```

En la salida aparecerá una sección similar a:

```text
Annotations:  curso.kubernetes.io/descripcion: Frontend de ejemplo para practicar labels y annotations
              curso.kubernetes.io/modulo: 4
              curso.kubernetes.io/nota: Entorno usado para practicar Namespaces, Labels y Annotations
```

Las Annotations sirven para guardar metadatos descriptivos o información usada por herramientas externas. No se utilizan para seleccionar recursos con `-l`. Para seleccionar recursos se usan Labels.

Ejemplos de información adecuada para Annotations:

- Descripción del recurso.
- Responsable o notas operativas.
- Enlace a documentación.
- Fecha o motivo de un cambio.
- Configuración usada por herramientas externas.

Ejemplos de información adecuada para Labels:

- `app=portal`
- `tier=frontend`
- `entorno=produccion`
- `equipo=plataforma`

## 7. Cambiar temporalmente el Namespace por defecto

Configuramos `m4-dev` como Namespace por defecto del contexto actual:

```bash
kubectl config set-context --current --namespace=m4-dev
```

Comprobamos el contexto:

```bash
kubectl config get-contexts
```

Ahora este comando consulta `m4-dev` sin indicar `-n`:

```bash
kubectl get pods
```

Salida esperada aproximada:

```text
NAME                               READY   STATUS    RESTARTS   AGE
portal-backend-xxxxxxxxxx-yyyyy    1/1     Running   0          5m
portal-frontend-xxxxxxxxxx-aaaaa   1/1     Running   0          5m
portal-frontend-xxxxxxxxxx-bbbbb   1/1     Running   0          5m
```

Esto es cómodo cuando vamos a trabajar durante un rato en un mismo Namespace. El riesgo es olvidarse de volver al Namespace habitual y ejecutar comandos contra el entorno equivocado.

Restauramos el Namespace por defecto:

```bash
kubectl config set-context --current --namespace=default
```

Comprobamos de nuevo:

```bash
kubectl config get-contexts
```

## 8. Limpieza

Eliminamos los Namespaces:

```bash
kubectl delete namespace m4-dev
kubectl delete namespace m4-prod
```

Al borrar un Namespace, Kubernetes elimina también los recursos namespaced que contiene.

Comprobamos:

```bash
kubectl get namespaces
```

Los Namespaces `m4-dev` y `m4-prod` ya no deberían aparecer.
