# Ejercicio 01 - Troubleshooting de volúmenes, ConfigMaps y Secrets

## Contexto

El equipo de plataforma ha desplegado una pequeña aplicación web llamada **reporting-web**.

La aplicación debería cumplir lo siguiente:

- Usar un `PersistentVolumeClaim` para almacenar el contenido servido por Nginx.
- Generar un archivo `index.html` desde un `initContainer`.
- Montar una configuración desde un `ConfigMap`.
- Leer credenciales desde un `Secret`.
- Exponer la aplicación mediante un `Service` interno de tipo `ClusterIP`.

Sin embargo, el despliegue tiene varios errores intencionados. Tu objetivo es diagnosticar los problemas y corregirlos aplicando los manifiestos disponibles en la carpeta `yaml/fix`.

> No debes escribir manifiestos YAML desde cero. Todos los archivos necesarios están incluidos.

---

## Estructura del ejercicio

```text
Ejercicio01/
├── yaml/
│   ├── 00-namespace.yaml
│   ├── 01-configmap-broken.yaml
│   ├── 02-secret-broken.yaml
│   ├── 03-pv.yaml
│   ├── 04-pvc-broken.yaml
│   ├── 05-deployment-broken.yaml
│   ├── 06-service.yaml
│   ├── 07-debug-pod.yaml
│   ├── 90-cleanup.yaml
│   └── fix/
│       ├── 01-pvc-fixed.yaml
│       ├── 02-configmap-fixed.yaml
│       └── 03-secret-fixed.yaml
├── Ejercicio01_Enunciado.md
└── Ejercicio01_Solucion.md
```

---

## Objetivos

Al finalizar el ejercicio deberías ser capaz de:

- Diagnosticar un `PersistentVolumeClaim` en estado `Pending`.
- Relacionar un `PVC` con un `PV` disponible.
- Interpretar eventos de Pods con `kubectl describe`.
- Detectar errores de montaje de claves de `ConfigMap`.
- Detectar errores de claves inexistentes en `Secret`.
- Validar que una aplicación usa correctamente almacenamiento persistente y configuración externa.

---

## Paso 1 - Desplegar el escenario con errores

Sitúate en la carpeta del ejercicio y aplica los manifiestos:

```bash
kubectl apply -f yaml/00-namespace.yaml
kubectl apply -f yaml/01-configmap-broken.yaml
kubectl apply -f yaml/02-secret-broken.yaml
kubectl apply -f yaml/03-pv.yaml
kubectl apply -f yaml/04-pvc-broken.yaml
kubectl apply -f yaml/05-deployment-broken.yaml
kubectl apply -f yaml/06-service.yaml
kubectl apply -f yaml/07-debug-pod.yaml
```

Comprueba el estado general:

```bash
kubectl get all -n m7-troubleshooting
kubectl get pv
kubectl get pvc -n m7-troubleshooting
```

---

## Paso 2 - Diagnosticar el primer problema

El Pod de la aplicación no debería arrancar correctamente.

Empieza investigando el estado del almacenamiento:

```bash
kubectl get pv
kubectl get pvc -n m7-troubleshooting
kubectl describe pvc reporting-data -n m7-troubleshooting
```

Preguntas:

1. ¿En qué estado está el `PVC`?
2. ¿Hay algún `PV` disponible?
3. ¿Por qué el `PVC` no se enlaza con el `PV`?
4. ¿Qué campo del `PVC` impide el enlace?

Cuando identifiques el problema, corrígelo usando el manifiesto correspondiente de la carpeta `yaml/fix`.

---

## Paso 3 - Diagnosticar el segundo problema

Después de corregir el `PVC`, revisa de nuevo el estado de los Pods:

```bash
kubectl get pods -n m7-troubleshooting
kubectl describe pod -n m7-troubleshooting -l app=reporting-web
```

Busca los eventos asociados al Pod.

Preguntas:

1. ¿Qué volumen no se puede montar correctamente?
2. ¿Qué objeto de Kubernetes está implicado?
3. ¿Qué clave falta?
4. ¿El problema está en el `Deployment` o en el `ConfigMap`?

Corrige el problema usando el manifiesto correspondiente de `yaml/fix`.

---

## Paso 4 - Diagnosticar el tercer problema

Tras corregir el `ConfigMap`, revisa otra vez el estado del Pod:

```bash
kubectl get pods -n m7-troubleshooting
kubectl describe pod -n m7-troubleshooting -l app=reporting-web
```

Preguntas:

1. ¿El Pod llega a ejecutar el contenedor principal?
2. ¿Qué error aparece ahora?
3. ¿Qué `Secret` está implicado?
4. ¿Qué clave falta dentro del `Secret`?

Corrige el problema usando el manifiesto correspondiente de `yaml/fix`.

---

## Paso 5 - Validar la solución

Cuando todo esté corregido, comprueba que el Pod está en estado `Running`:

```bash
kubectl get pods -n m7-troubleshooting -o wide
```

Comprueba que el `PVC` está enlazado:

```bash
kubectl get pvc -n m7-troubleshooting
kubectl get pv
```

Comprueba que el archivo generado por el `initContainer` existe dentro del volumen persistente:

```bash
POD=$(kubectl get pod -n m7-troubleshooting -l app=reporting-web -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n m7-troubleshooting "$POD" -c nginx -- cat /usr/share/nginx/html/index.html
```

Comprueba que el `ConfigMap` se ha montado correctamente:

```bash
kubectl exec -n m7-troubleshooting "$POD" -c nginx -- ls -l /etc/reporting
kubectl exec -n m7-troubleshooting "$POD" -c nginx -- cat /etc/reporting/application.properties
```

Comprueba el acceso mediante el `Service` desde el Pod de debug:

```bash
kubectl exec -n m7-troubleshooting debug-shell -- curl -s http://reporting-web
```

---

## Paso 6 - Preguntas finales

Responde brevemente:

1. ¿Por qué un `PVC` puede quedar en estado `Pending` aunque exista un `PV` disponible?
2. ¿Qué diferencia hay entre montar un `ConfigMap` completo y montar claves concretas usando `items`?
3. ¿Qué ocurre si un Pod referencia una clave de `Secret` que no existe?
4. ¿El contenido generado por el `initContainer` se perdería si se elimina y recrea el Pod? ¿Por qué?
5. ¿Usarías `hostPath` para almacenamiento persistente en producción? Justifica la respuesta.

---

## Limpieza

Para eliminar los recursos del ejercicio:

```bash
kubectl delete namespace m7-troubleshooting
kubectl delete pv pv-reporting-local
```

Si quieres limpiar también los datos creados en el nodo donde se haya ejecutado el Pod:

```bash
sudo rm -rf /tmp/k8s-m7-reporting
```
