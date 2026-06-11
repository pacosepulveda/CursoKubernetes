# Solución — Troubleshooting en Kubernetes: CrashLoopBackOff, OOMKilled y dependencias de configuración

## Validación previa del laboratorio

Este laboratorio es compatible con un clúster Kubernetes desplegado con `kubeadm` sobre AWS con:

- 1 nodo control plane / master.
- 2 nodos worker.
- Instancias EC2 `t3.medium`.
- CNI Calico.
- Acceso con `kubectl`.

Los recursos del laboratorio son pequeños y los `requests` de memoria son bajos, por lo que no deberían saturar dos workers `t3.medium`.

Puntos importantes:

- El nodo master normalmente tendrá taint de control plane y no ejecutará Pods de aplicación.
- Los Pods se programarán en los workers.
- Calico no debería afectar a estos escenarios porque no dependen de conectividad entre Pods.
- Se necesita salida a Internet desde los nodos para descargar las imágenes `busybox:1.36` y `python:3.11-alpine`.
- `metrics-server` no es obligatorio. Solo sería necesario para usar `kubectl top`.

> Nota importante sobre el escenario 2: el archivo original de corrección aumentaba el límite de memoria, pero el proceso finalizaba a los 30 segundos. En un `Deployment`, un contenedor que termina también se reinicia porque la política por defecto es `Always`. Para que el ejercicio quede estable, se recomienda usar una versión corregida en la que el proceso permanezca vivo.

---

## Preparación

Crear el namespace:

```bash
kubectl apply -f ns.yaml
```

Comprobar el estado inicial:

```bash
kubectl get ns lab-troubleshooting
kubectl get pods -n lab-troubleshooting
```

---

# Escenario 1 — CrashLoopBackOff por fallo de aplicación

## 1. Desplegar el recurso problemático

```bash
kubectl apply -f scenario1-bug-deploy.yaml
```

Comprobar el Deployment:

```bash
kubectl get deploy -n lab-troubleshooting
```

Comprobar los Pods:

```bash
kubectl get pods -n lab-troubleshooting
```

También se puede observar en tiempo real:

```bash
kubectl get pods -n lab-troubleshooting -w
```

El Pod asociado al Deployment `app-con-bug` acabará entrando en estado similar a:

```text
CrashLoopBackOff
```

---

## 2. Identificar el Pod

```bash
POD=$(kubectl get pods -n lab-troubleshooting -l app=app-con-bug -o jsonpath='{.items[0].metadata.name}')
echo $POD
```

---

## 3. Describir el Pod

```bash
kubectl describe pod $POD -n lab-troubleshooting
```

En la salida hay que revisar especialmente:

- Estado del contenedor.
- Número de reinicios.
- Último estado terminado.
- Código de salida.
- Eventos.

Para ver solo los eventos:

```bash
kubectl describe pod $POD -n lab-troubleshooting | sed -n '/Events:/,$p'
```

El fallo se debe a que el proceso principal del contenedor termina con error.

En el manifiesto problemático aparece este comando:

```yaml
command: ["sh", "-c", "echo 'Crasheando intencionadamente'; exit 1"]
```

El contenedor imprime un mensaje y después termina con código de salida `1`.

---

## 4. Consultar logs

En un `CrashLoopBackOff`, el contenedor se reinicia continuamente. Por eso puede ser necesario consultar los logs del intento anterior:

```bash
kubectl logs $POD -n lab-troubleshooting --previous
```

La salida esperada será similar a:

```text
Crasheando intencionadamente
```

También se pueden consultar los logs actuales:

```bash
kubectl logs $POD -n lab-troubleshooting
```

Pero, dependiendo del momento exacto, puede que el contenedor ya haya terminado y los logs más útiles estén en `--previous`.

---

## 5. Causa raíz

La causa raíz es un fallo en el comando de arranque de la aplicación.

No es un problema de:

- Imagen inexistente.
- Scheduling.
- Red.
- Calico.
- Namespace.
- Permisos.
- Recursos de memoria.

El contenedor arranca, ejecuta su comando y termina voluntariamente con error.

---

## 6. Aplicar la corrección

Aplicar el manifiesto corregido:

```bash
kubectl apply -f scenario1-fix-deploy.yaml
```

Este manifiesto sustituye el comando problemático por uno que deja el contenedor vivo:

```yaml
command: ["sh", "-c", "echo 'Iniciado correctamente'; sleep 3600"]
```

---

## 7. Verificar

Comprobar el rollout:

```bash
kubectl rollout status deploy/app-con-bug -n lab-troubleshooting
```

Comprobar el Pod:

```bash
kubectl get pods -n lab-troubleshooting -l app=app-con-bug
```

El estado esperado es:

```text
Running
```

Comprobar los logs:

```bash
POD=$(kubectl get pods -n lab-troubleshooting -l app=app-con-bug -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD -n lab-troubleshooting
```

Salida esperada:

```text
Iniciado correctamente
```

---

# Escenario 2 — CrashLoopBackOff por OOMKilled

## 1. Desplegar el recurso problemático

```bash
kubectl apply -f scenario2-oom-deploy.yaml
```

Observar el estado:

```bash
kubectl get pods -n lab-troubleshooting -l app=app-oom -w
```

El Pod acabará entrando en un patrón de reinicios.

---

## 2. Identificar el Pod

```bash
POD=$(kubectl get pods -n lab-troubleshooting -l app=app-oom -o jsonpath='{.items[0].metadata.name}')
echo $POD
```

---

## 3. Describir el Pod

```bash
kubectl describe pod $POD -n lab-troubleshooting
```

En la salida hay que buscar una sección similar a:

```text
Last State:     Terminated
Reason:         OOMKilled
```

También puede verse el código de salida típico de un proceso terminado por falta de memoria:

```text
Exit Code:      137
```

---

## 4. Revisar la configuración de recursos

El manifiesto problemático contiene:

```yaml
resources:
  requests:
    memory: "32Mi"
  limits:
    memory: "64Mi"
```

Sin embargo, el proceso intenta reservar aproximadamente 200 MiB:

```bash
python -c "import time; x=bytearray(200*1024*1024); time.sleep(30)"
```

El límite de memoria es inferior a la memoria que intenta reservar el proceso. Por eso el kernel termina el proceso dentro del contenedor y Kubernetes informa de `OOMKilled`.

---

## 5. Causa raíz

La causa raíz es un límite de memoria demasiado bajo para el consumo real de la aplicación.

Este escenario sí produce reinicios, pero la causa no es la misma que en el escenario 1:

| Escenario | Estado | Causa |
|---|---|---|
| Escenario 1 | CrashLoopBackOff | El proceso termina con `exit 1` |
| Escenario 2 | CrashLoopBackOff / OOMKilled | El proceso supera el límite de memoria |

---

## 6. Corrección recomendada

Hay dos formas conceptuales de corregir este problema:

1. Reducir el consumo de memoria de la aplicación.
2. Aumentar el límite de memoria del contenedor.

En este laboratorio aplicaremos la segunda opción.

Se recomienda usar este contenido para el archivo `scenario2-fix-patch.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-oom
  namespace: lab-troubleshooting
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-oom
  template:
    metadata:
      labels:
        app: app-oom
    spec:
      containers:
        - name: app
          image: python:3.11-alpine
          command:
            - sh
            - -c
            - |
              python -c "import time; x=bytearray(200*1024*1024); print('Memoria reservada correctamente'); time.sleep(3600)"
          resources:
            requests:
              memory: "64Mi"
            limits:
              memory: "256Mi"
```

Aplicar la corrección:

```bash
kubectl apply -f scenario2-fix-patch.yaml
```

---

## 7. Verificar

Comprobar el rollout:

```bash
kubectl rollout status deploy/app-oom -n lab-troubleshooting
```

Comprobar el Pod:

```bash
kubectl get pods -n lab-troubleshooting -l app=app-oom
```

El estado esperado es:

```text
Running
```

Comprobar los logs:

```bash
POD=$(kubectl get pods -n lab-troubleshooting -l app=app-oom -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD -n lab-troubleshooting
```

Salida esperada:

```text
Memoria reservada correctamente
```

Si hay `metrics-server`, se puede revisar el consumo:

```bash
kubectl top pod -n lab-troubleshooting
```

---

# Escenario 3 — CreateContainerConfigError por ConfigMap inexistente

## 1. Desplegar el recurso problemático

```bash
kubectl apply -f scenario3-config-dependency-pod.yaml
```

Comprobar el estado:

```bash
kubectl get pod app-dep -n lab-troubleshooting
```

El estado esperado es:

```text
CreateContainerConfigError
```

---

## 2. Analizar el fallo

Describir el Pod:

```bash
kubectl describe pod app-dep -n lab-troubleshooting
```

Revisar los eventos:

```bash
kubectl describe pod app-dep -n lab-troubleshooting | sed -n '/Events:/,$p'
```

Aparecerá un mensaje indicando que falta el ConfigMap referenciado.

El Pod contiene esta variable de entorno:

```yaml
env:
  - name: MI_CONFIG
    valueFrom:
      configMapKeyRef:
        name: config-inexistente
        key: data.key
```

El contenedor necesita leer la clave `data.key` desde un ConfigMap llamado `config-inexistente`.

Como ese ConfigMap no existe todavía, Kubernetes no puede construir la configuración del contenedor y el Pod queda en `CreateContainerConfigError`.

---

## 3. Diferencia con CrashLoopBackOff

Este escenario no es un `CrashLoopBackOff`.

La diferencia es importante:

| Estado | ¿Arranca el contenedor? | ¿Tiene sentido mirar logs? | Causa típica |
|---|---:|---:|---|
| CrashLoopBackOff | Sí, pero termina | Sí | Proceso falla después de arrancar |
| OOMKilled | Sí, pero se mata por memoria | A veces | Supera el límite de memoria |
| CreateContainerConfigError | No | Normalmente no | Falta configuración necesaria |

En este caso, el contenedor no llega a arrancar, por lo que los logs no son la fuente principal de diagnóstico. La información clave está en los eventos del Pod.

---

## 4. Aplicar la corrección

Crear el ConfigMap esperado:

```bash
kubectl apply -f scenario3-configmap-fix.yaml
```

El archivo contiene:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-inexistente
  namespace: lab-troubleshooting
data:
  data.key: "valor-de-ejemplo"
```

---

## 5. Verificar

Observar la transición del Pod:

```bash
kubectl get pod app-dep -n lab-troubleshooting -w
```

El Pod debería pasar a:

```text
Running
```

Comprobar los detalles:

```bash
kubectl describe pod app-dep -n lab-troubleshooting
```

Comprobar logs:

```bash
kubectl logs app-dep -n lab-troubleshooting
```

Salida esperada:

```text
Arrancando...
```

---

# Verificación final del laboratorio

Comprobar todos los recursos:

```bash
kubectl get all -n lab-troubleshooting
```

Estado esperado:

- `app-con-bug`: corregida y en ejecución.
- `app-oom`: corregida y en ejecución.
- `app-dep`: en ejecución tras crear el ConfigMap.

Comprobar Pods:

```bash
kubectl get pods -n lab-troubleshooting -o wide
```

Los Pods deberían estar en estado `Running`.

---

# Limpieza

Eliminar el namespace completo:

```bash
kubectl delete ns lab-troubleshooting --ignore-not-found=true
```

Comprobar que se ha eliminado:

```bash
kubectl get ns lab-troubleshooting
```

Si el namespace ya no existe, el laboratorio ha quedado limpio.
