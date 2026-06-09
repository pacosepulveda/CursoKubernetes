# Solución - Ejercicio 1 - Diagnóstico básico de Pods con `kubectl`

## 1. Preparar el entorno

Comprobamos primero que `kubectl` puede comunicarse con el clúster:

```bash
kubectl get nodes
kubectl get pods -A
```

Creamos un Namespace para aislar el ejercicio:

```bash
kubectl create namespace m4-diagnostico
```

Aplicamos el manifiesto YAML en ese Namespace:

```bash
kubectl apply -n m4-diagnostico -f yaml/modulo4-ejercicio01-diagnostico.yaml
```

El archivo YAML no incluye un Namespace fijo. Esto permite reutilizarlo en cualquier Namespace usando la opción `-n`.

## 2. Consultar el estado general de los Pods

Ejecutamos:

```bash
kubectl get pods -n m4-diagnostico
```

Salida esperada aproximada:

```text
NAME               READY   STATUS             RESTARTS      AGE
m4-crashloop       0/1     CrashLoopBackOff   2             60s
m4-image-error     0/1     ImagePullBackOff   0             60s
m4-log-generator   1/1     Running            0             60s
m4-web-ok          1/1     Running            0             60s
```

Los estados pueden variar durante los primeros segundos. Por ejemplo, `m4-crashloop` puede aparecer temporalmente como `Error`, `Running` o `CrashLoopBackOff`, porque Kubernetes intenta reiniciarlo.

Mostramos más información:

```bash
kubectl get pods -n m4-diagnostico -o wide
```

Esta salida permite ver el nodo donde se ejecuta cada Pod. Los Pods que no han podido arrancar correctamente pueden no tener IP asignada o no mostrar toda la información esperada.

## 3. Inspeccionar el Pod con `CrashLoopBackOff`

Ejecutamos:

```bash
kubectl describe pod m4-crashloop -n m4-diagnostico
```

En la salida debemos revisar especialmente estas secciones:

```text
Containers:
  worker:
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
    Restart Count:  2
```

La sección `Events` suele mostrar mensajes similares a:

```text
Back-off restarting failed container worker in pod m4-crashloop
```

Interpretación:

- Kubernetes ha creado el Pod correctamente.
- El contenedor se inicia, ejecuta su comando y termina con código de salida `1`.
- Como la política de reinicio es `Always`, Kubernetes intenta reiniciarlo.
- Tras varios fallos, aparece el estado `CrashLoopBackOff`.

## 4. Consultar logs actuales y logs anteriores

Consultamos los logs del contenedor:

```bash
kubectl logs m4-crashloop -n m4-diagnostico -c worker
```

Salida esperada:

```text
Iniciando proceso batch
Simulando un error de aplicacion
```

Cuando el contenedor ya se ha reiniciado al menos una vez, podemos consultar los logs de la instancia anterior:

```bash
kubectl logs --previous m4-crashloop -n m4-diagnostico -c worker
```

`--previous` es útil cuando un contenedor se ha reiniciado. Permite ver los logs de la ejecución anterior, que normalmente contienen la causa del fallo.

## 5. Consultar logs de un Pod que genera salida continua

Ejecutamos:

```bash
kubectl logs m4-log-generator -n m4-diagnostico -c logger --tail=10
```

Salida esperada aproximada:

```text
stdout - mensaje informativo numero 5
stderr - mensaje de advertencia numero 5
stdout - mensaje informativo numero 6
stderr - mensaje de advertencia numero 6
...
```

Para seguir los logs en tiempo real:

```bash
kubectl logs -f m4-log-generator -n m4-diagnostico -c logger
```

El contenedor escribe una nueva pareja de mensajes aproximadamente cada 3 segundos. Aparecen mensajes enviados a `stdout` y a `stderr`; `kubectl logs` muestra ambos flujos.

Para salir, pulsamos `Ctrl+C`.

## 6. Ejecutar comandos dentro del contenedor correcto

Ejecutamos un comando puntual dentro del contenedor `web`:

```bash
kubectl exec -n m4-diagnostico m4-web-ok -c web -- nginx -v
```

Salida esperada aproximada:

```text
nginx version: nginx/1.27.x
```

Abrimos una shell interactiva:

```bash
kubectl exec -it -n m4-diagnostico m4-web-ok -c web -- /bin/sh
```

Dentro del contenedor:

```sh
hostname
ls /usr/share/nginx/html
exit
```

Diferencia importante:

- Con `kubectl exec ... -- comando` ejecutamos una orden puntual y volvemos a nuestra terminal.
- Con `kubectl exec -it ... -- /bin/sh` abrimos una sesión interactiva dentro del contenedor.

No es buena práctica instalar herramientas o modificar archivos manualmente dentro de un contenedor como solución definitiva. Esos cambios son efímeros y se pierden cuando el contenedor se recrea. La solución correcta es modificar la imagen o el manifiesto y volver a desplegar.

## 7. Diagnosticar el Pod con error de imagen

Ejecutamos:

```bash
kubectl describe pod m4-image-error -n m4-diagnostico
```

En la salida veremos eventos parecidos a:

```text
Failed to pull image "nginx:version-inexistente-modulo4"
Error: ErrImagePull
Back-off pulling image "nginx:version-inexistente-modulo4"
Error: ImagePullBackOff
```

Interpretación:

- Kubernetes acepta el Pod.
- El kubelet del nodo intenta descargar la imagen.
- La imagen indicada no existe.
- El Pod queda en estado `ImagePullBackOff`.

El problema no está en la API de Kubernetes ni en el scheduler. El problema está en la referencia de imagen configurada en el manifiesto.

## 8. Limpieza

Eliminamos los recursos:

```bash
kubectl delete -n m4-diagnostico -f yaml/modulo4-ejercicio01-diagnostico.yaml
kubectl delete namespace m4-diagnostico
```

Comprobamos que no quedan recursos del ejercicio:

```bash
kubectl get pods -n m4-diagnostico
```

Si el Namespace ya se ha eliminado, Kubernetes devolverá un error indicando que no existe. Es normal.
