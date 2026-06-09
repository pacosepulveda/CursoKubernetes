# Ejercicio 2 - Namespaces, Labels y Annotations

## Objetivo

Practicar la organización de recursos de Kubernetes usando Namespaces, Labels y Annotations.

Este ejercicio se centra únicamente en conceptos del módulo 4:

- Creación y consulta de Namespaces.
- Despliegue del mismo manifiesto YAML en distintos Namespaces.
- Uso de Labels para clasificar y seleccionar recursos.
- Uso de Annotations para añadir metadatos descriptivos.
- Consulta de recursos con `kubectl get`, `kubectl describe` y selectores.

## Archivos necesarios

Copia en tu nodo de administración el siguiente archivo:

- `yaml/modulo4-ejercicio02-aplicacion.yaml`

## Escenario

Vas a simular una pequeña aplicación llamada `portal` desplegada en dos entornos separados:

- `m4-dev`: entorno de desarrollo.
- `m4-prod`: entorno de producción.

El mismo archivo YAML se aplicará en ambos Namespaces. Esto permite comprobar que los nombres de los objetos solo tienen que ser únicos dentro de un Namespace, no en todo el clúster.

## Preparación

Comprueba el acceso al clúster:

```bash
kubectl get nodes
```

Crea los Namespaces:

```bash
kubectl create namespace m4-dev
kubectl create namespace m4-prod
```

Despliega la aplicación en ambos Namespaces:

```bash
kubectl apply -n m4-dev -f yaml/modulo4-ejercicio02-aplicacion.yaml
kubectl apply -n m4-prod -f yaml/modulo4-ejercicio02-aplicacion.yaml
```

## Tareas

### 1. Comprobar los Namespaces

Lista los Namespaces del clúster:

```bash
kubectl get namespaces
```

Comprueba los Deployments de cada Namespace:

```bash
kubectl get deployments -n m4-dev
kubectl get deployments -n m4-prod
```

Responde:

- ¿Existen Deployments con el mismo nombre en ambos Namespaces?
- ¿Por qué Kubernetes lo permite?

### 2. Consultar Pods y Labels

Lista los Pods de cada Namespace mostrando sus Labels:

```bash
kubectl get pods -n m4-dev --show-labels
kubectl get pods -n m4-prod --show-labels
```

Responde:

- ¿Qué Labels identifican a todos los Pods de la aplicación `portal`?
- ¿Qué Label permite distinguir frontend y backend?

### 3. Filtrar recursos por Labels

Lista solo los Pods frontend en `m4-dev`:

```bash
kubectl get pods -n m4-dev -l app=portal,tier=frontend
```

Lista solo los Pods backend en `m4-prod`:

```bash
kubectl get pods -n m4-prod -l app=portal,tier=backend
```

Lista todos los Pods de la aplicación `portal` en todos los Namespaces:

```bash
kubectl get pods -A -l app=portal
```

Responde:

- ¿Cuántos Pods frontend hay en cada Namespace?
- ¿Cuántos Pods backend hay en cada Namespace?
- ¿Qué ventaja tiene usar Labels frente a recordar nombres concretos de Pods?

### 4. Añadir una Label manualmente

Añade una Label que identifique el entorno en todos los Deployments de `m4-dev`:

```bash
kubectl label deployment -n m4-dev -l app=portal entorno=desarrollo
```

Añade una Label equivalente en `m4-prod`:

```bash
kubectl label deployment -n m4-prod -l app=portal entorno=produccion
```

Comprueba el resultado:

```bash
kubectl get deployments -n m4-dev --show-labels
kubectl get deployments -n m4-prod --show-labels
```

Responde:

- ¿Se han etiquetado los Deployments?
- ¿Se han etiquetado automáticamente los Pods que ya existían?
- ¿Qué diferencia hay entre etiquetar un Deployment y etiquetar los Pods creados por ese Deployment?

### 5. Añadir una Annotation descriptiva

Añade una Annotation a los Deployments del entorno de desarrollo:

```bash
kubectl annotate deployment -n m4-dev -l app=portal curso.kubernetes.io/nota="Entorno usado para practicar Namespaces, Labels y Annotations"
```

Consulta uno de los Deployments con `describe`:

```bash
kubectl describe deployment portal-frontend -n m4-dev
```

Responde:

- ¿Dónde aparecen las Annotations?
- ¿Se usan las Annotations para seleccionar recursos con `-l`?
- ¿Qué tipo de información guardarías en una Annotation y no en una Label?

### 6. Cambiar temporalmente el Namespace por defecto

Cambia el Namespace por defecto del contexto actual a `m4-dev`:

```bash
kubectl config set-context --current --namespace=m4-dev
```

Comprueba el contexto actual:

```bash
kubectl config get-contexts
```

Ahora ejecuta:

```bash
kubectl get pods
```

Vuelve a dejar el Namespace por defecto en `default`:

```bash
kubectl config set-context --current --namespace=default
```

Responde:

- ¿Qué ventaja tiene configurar temporalmente un Namespace por defecto?
- ¿Qué riesgo puede tener olvidarse de volver al Namespace habitual?

## Limpieza

Elimina los dos Namespaces:

```bash
kubectl delete namespace m4-dev
kubectl delete namespace m4-prod
```

Comprueba que se han eliminado:

```bash
kubectl get namespaces
```
