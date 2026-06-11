# Solución — Scheduling con Jobs, CronJobs y Node Affinity

## 1. Revisar el estado general

Comenzamos revisando los recursos del namespace:

```bash
kubectl get all -n cka-scheduling-lab
```

Podemos ver que el `Job` `report-once` no completa y que hay pods en estado `Pending`.

```bash
kubectl get jobs -n cka-scheduling-lab
kubectl get pods -n cka-scheduling-lab -o wide
```

Una salida típica sería:

```text
NAME                    READY   STATUS    RESTARTS   AGE   IP       NODE
report-once-abcde       0/1     Pending   0          2m    <none>   <none>
```

El detalle importante es que el pod no tiene nodo asignado. Eso indica que el problema ocurre antes de arrancar el contenedor.

## 2. Describir el pod pendiente

Obtenemos el pod del `Job`:

```bash
POD=$(kubectl get pods -n cka-scheduling-lab -l job-name=report-once -o jsonpath='{.items[0].metadata.name}')
echo $POD
```

Lo describimos:

```bash
kubectl describe pod "$POD" -n cka-scheduling-lab
```

En la sección `Events` aparece el diagnóstico principal:

```text
Warning  FailedScheduling  default-scheduler  0/3 nodes are available:
3 node(s) didn't match Pod's node affinity/selector.
```

El mensaje indica que el scheduler no puede colocar el pod porque ningún nodo cumple la afinidad requerida.

Esto descarta problemas como:

* Imagen inexistente.
* CrashLoopBackOff.
* Error de permisos dentro del contenedor.
* Fallo del comando del contenedor.

El contenedor ni siquiera ha llegado a arrancar.

## 3. Revisar la afinidad del Job

Inspeccionamos el manifiesto efectivo del `Job`:

```bash
kubectl get job report-once -n cka-scheduling-lab -o yaml
```

La parte relevante es:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: cka.formacion.io/pool
              operator: In
              values:
                - batch
```

Esto significa que el pod solo puede ejecutarse en nodos que tengan esta etiqueta:

```text
cka.formacion.io/pool=batch
```

Como se usa `requiredDuringSchedulingIgnoredDuringExecution`, la condición es obligatoria. Si ningún nodo la cumple, el pod se queda en `Pending`.

## 4. Comprobar etiquetas de nodos

Ejecutamos:

```bash
kubectl get nodes -L cka.formacion.io/pool
```

Salida esperada antes del fix:

```text
NAME       STATUS   ROLES           AGE   VERSION   POOL
master-1   Ready    control-plane   ...   v1.35.5
worker-1   Ready    <none>          ...   v1.35.5
worker-2   Ready    <none>          ...   v1.35.5
```

No aparece ningún nodo con valor `batch` en la columna `POOL`.

También se puede comprobar con:

```bash
kubectl get nodes --show-labels | grep cka.formacion.io/pool
```

Si no devuelve nada, la etiqueta no existe en ningún nodo.

## 5. Corregir la causa raíz

La corrección mínima es etiquetar un worker como nodo apto para cargas batch.

Primero elegimos un worker. Por ejemplo:

```bash
kubectl get nodes
```

Después etiquetamos uno de los workers:

```bash
kubectl label node <WORKER_ELEGIDO> cka.formacion.io/pool=batch
```

Ejemplo:

```bash
kubectl label node worker-1 cka.formacion.io/pool=batch
```

Verificamos:

```bash
kubectl get nodes -L cka.formacion.io/pool
```

Salida esperada:

```text
NAME       STATUS   ROLES           AGE   VERSION   POOL
master-1   Ready    control-plane   ...   v1.35.5
worker-1   Ready    <none>          ...   v1.35.5   batch
worker-2   Ready    <none>          ...   v1.35.5
```

## 6. Verificar que el Job se programa

Tras añadir la etiqueta, el scheduler ya puede asignar el pod pendiente.

```bash
kubectl get pods -n cka-scheduling-lab -o wide -w
```

El pod debería pasar por estados similares a:

```text
Pending -> ContainerCreating -> Running -> Completed
```

Después comprobamos el `Job`:

```bash
kubectl get job report-once -n cka-scheduling-lab
```

Resultado esperado:

```text
NAME          STATUS     COMPLETIONS   DURATION   AGE
report-once   Complete   1/1           ...        ...
```

## 7. Diagnóstico del CronJob

Revisamos el `CronJob`:

```bash
kubectl get cronjob -n cka-scheduling-lab
kubectl describe cronjob report-cron -n cka-scheduling-lab
```

El `CronJob` tiene también la misma afinidad obligatoria:

```yaml
cka.formacion.io/pool=batch
```

Antes del fix, su primer `Job` podía quedar activo pero con el pod en `Pending`.

Como el `CronJob` tiene:

```yaml
concurrencyPolicy: Forbid
```

Kubernetes no lanza una nueva ejecución mientras la anterior siga activa.

Esto puede dar la impresión de que el `CronJob` no funciona, pero la causa raíz sigue siendo la misma: la ejecución anterior nunca pudo programarse.

## 8. Verificar el CronJob después del fix

Tras etiquetar el nodo, el `Job` pendiente del `CronJob` puede ejecutarse.

Comprobamos:

```bash
kubectl get jobs -n cka-scheduling-lab
kubectl get pods -n cka-scheduling-lab -o wide
```

Después de uno o dos minutos:

```bash
kubectl get jobs -n cka-scheduling-lab
```

Deberían aparecer nuevas ejecuciones del `CronJob`.

También se puede revisar:

```bash
kubectl describe cronjob report-cron -n cka-scheduling-lab
```

## 9. Conclusión

La causa raíz era una afinidad de nodo obligatoria que ningún nodo cumplía.

El `Job` no fallaba por su comando ni por la imagen. El pod estaba en `Pending` porque el scheduler no encontraba ningún nodo con:

```text
cka.formacion.io/pool=batch
```

El `CronJob` parecía tener un segundo problema, pero en realidad era un efecto secundario:

1. El `CronJob` creó un `Job`.
2. El pod de ese `Job` quedó en `Pending`.
3. El `Job` siguió activo.
4. Como `concurrencyPolicy` era `Forbid`, el `CronJob` no creó nuevas ejecuciones mientras la anterior seguía activa.

## Comandos clave

```bash
kubectl get pods -n cka-scheduling-lab -o wide
kubectl describe pod <POD> -n cka-scheduling-lab
kubectl get job -n cka-scheduling-lab
kubectl get cronjob -n cka-scheduling-lab
kubectl describe cronjob report-cron -n cka-scheduling-lab
kubectl get nodes -L cka.formacion.io/pool
kubectl label node <WORKER> cka.formacion.io/pool=batch
```

## Limpieza

```bash
kubectl delete ns cka-scheduling-lab
kubectl label node <WORKER> cka.formacion.io/pool-
```
