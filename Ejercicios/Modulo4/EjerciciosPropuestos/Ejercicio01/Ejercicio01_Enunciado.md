# Ejercicio 1 - Diagnóstico básico de Pods con `kubectl`

## Objetivo

Practicar el uso de `kubectl` para inspeccionar el estado de varios Pods, consultar información detallada, revisar logs y ejecutar comandos dentro de un contenedor.

- Acceso al clúster mediante `kubectl`.
- Consulta de recursos con `kubectl get`.
- Inspección detallada con `kubectl describe`.
- Consulta de logs con `kubectl logs`.
- Ejecución de comandos dentro de un contenedor con `kubectl exec`.
- Identificación de estados como `Running`, `CrashLoopBackOff` e `ImagePullBackOff`.

## Archivos necesarios

Copia en tu nodo de administración el siguiente archivo:

- `yaml/modulo4-ejercicio01-diagnostico.yaml`

## Preparación

Comprueba que tienes acceso al clúster:

```bash
kubectl get nodes
kubectl get pods -A
```

Crea un Namespace específico para este ejercicio:

```bash
kubectl create namespace m4-diagnostico
```

Despliega los recursos del ejercicio en ese Namespace:

```bash
kubectl apply -n m4-diagnostico -f yaml/modulo4-ejercicio01-diagnostico.yaml
```

## Tareas

### 1. Consultar el estado general de los Pods

Lista los Pods del Namespace `m4-diagnostico`:

```bash
kubectl get pods -n m4-diagnostico
```

Después, muestra más información usando la salida ampliada:

```bash
kubectl get pods -n m4-diagnostico -o wide
```

Responde:

- ¿Qué Pods están en estado `Running`?
- ¿Qué Pod está en `CrashLoopBackOff`?
- ¿Qué Pod tiene un problema relacionado con la descarga de la imagen?
- ¿En qué nodo se está ejecutando cada Pod que haya podido arrancar?

### 2. Inspeccionar el Pod que falla por reinicios

Obtén información detallada del Pod `m4-crashloop`:

```bash
kubectl describe pod m4-crashloop -n m4-diagnostico
```

Localiza en la salida:

- Estado del contenedor.
- Número de reinicios.
- Motivo del fallo.
- Eventos del Pod.

### 3. Consultar logs del Pod que se reinicia

Consulta los logs actuales del Pod `m4-crashloop`:

```bash
kubectl logs m4-crashloop -n m4-diagnostico -c worker
```

Cuando el Pod tenga al menos un reinicio, consulta los logs de la instancia anterior del contenedor:

```bash
kubectl logs --previous m4-crashloop -n m4-diagnostico -c worker
```

Responde:

- ¿Qué mensaje aparece antes de que el contenedor finalice?
- ¿Por qué tiene sentido usar `--previous` en este caso?

### 4. Consultar logs de un Pod que genera salida continua

Consulta las últimas 10 líneas del Pod `m4-log-generator`:

```bash
kubectl logs m4-log-generator -n m4-diagnostico -c logger --tail=10
```

Sigue los logs en tiempo real:

```bash
kubectl logs -f m4-log-generator -n m4-diagnostico -c logger
```

Pulsa `Ctrl+C` para salir.

Responde:

- ¿Se muestran mensajes de salida estándar y de error?
- ¿Cada cuánto tiempo aparece una nueva línea aproximadamente?

### 5. Ejecutar un comando dentro de un contenedor

Ejecuta un comando dentro del contenedor `web` del Pod `m4-web-ok`:

```bash
kubectl exec -n m4-diagnostico m4-web-ok -c web -- nginx -v
```

Abre una shell interactiva dentro del contenedor:

```bash
kubectl exec -it -n m4-diagnostico m4-web-ok -c web -- /bin/sh
```

Dentro del contenedor, ejecuta:

```sh
hostname
ls /usr/share/nginx/html
exit
```

Responde:

- ¿Qué diferencia hay entre ejecutar un comando puntual con `kubectl exec` y abrir una shell interactiva con `-it`?
- ¿Por qué los cambios manuales dentro de un contenedor no deberían considerarse persistentes?

### 6. Diagnosticar el Pod con error de imagen

Inspecciona el Pod `m4-image-error`:

```bash
kubectl describe pod m4-image-error -n m4-diagnostico
```

Responde:

- ¿Qué estado muestra el Pod?
- ¿Qué eventos indican que Kubernetes no puede descargar la imagen?
- ¿El problema parece estar en Kubernetes o en la imagen indicada en el manifiesto?

## Limpieza

Elimina los recursos del ejercicio:

```bash
kubectl delete -n m4-diagnostico -f yaml/modulo4-ejercicio01-diagnostico.yaml
kubectl delete namespace m4-diagnostico
```
