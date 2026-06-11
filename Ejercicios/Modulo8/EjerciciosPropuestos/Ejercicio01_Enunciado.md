# Laboratorio — Troubleshooting de Scheduling con Jobs, CronJobs y Node Affinity

## Objetivo

Diagnosticar un problema de scheduling en Kubernetes relacionado con:

* `Job`
* `CronJob`
* `nodeAffinity`
* eventos de `FailedScheduling`
* etiquetas de nodos

Los manifiestos se aplican correctamente, pero las cargas de trabajo no se ejecutan como se espera.

## Escenario

El equipo de plataforma ha preparado un namespace para tareas batch.

Hay dos cargas de trabajo:

1. Un `Job` llamado `report-once`, que debería ejecutarse una vez y terminar correctamente.
2. Un `CronJob` llamado `report-cron`, que debería lanzar una ejecución periódica.

Ambas cargas deben ejecutarse en nodos preparados para cargas batch.

Sin embargo, el equipo informa de que:

* El `Job` no termina.
* Hay pods en estado `Pending`.
* El `CronJob` no parece generar ejecuciones nuevas después de un tiempo.
* El clúster tiene workers disponibles, pero las tareas batch no arrancan.

Tu tarea es diagnosticar la causa raíz y aplicar la corrección mínima necesaria.

## Despliegue inicial

Aplica los manifiestos:

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 10-report-job.yaml
kubectl apply -f 20-report-cronjob.yaml
```

Comprueba los recursos creados:

```bash
kubectl get all -n cka-scheduling-lab
```

## Tareas

### 1. Revisar el estado del Job

```bash
kubectl get job -n cka-scheduling-lab
kubectl get pods -n cka-scheduling-lab -o wide
```

Preguntas:

* ¿El `Job` ha completado?
* ¿El pod del `Job` está ejecutándose?
* ¿En qué estado está?

### 2. Analizar el pod en `Pending`

Obtén el nombre del pod del `Job`:

```bash
POD=$(kubectl get pods -n cka-scheduling-lab -l job-name=report-once -o jsonpath='{.items[0].metadata.name}')
echo $POD
```

Describe el pod:

```bash
kubectl describe pod "$POD" -n cka-scheduling-lab
```

Revisa especialmente la sección `Events`.

Preguntas:

* ¿Qué componente está rechazando el scheduling?
* ¿Qué mensaje aparece en los eventos?
* ¿El problema es de CPU, memoria, taints, afinidad o imagen?

### 3. Corregir el problema

Aplica la corrección mínima para que las cargas batch puedan programarse en uno de los workers.

Después verifica:

```bash
kubectl get pods -n cka-scheduling-lab -o wide
kubectl get jobs -n cka-scheduling-lab
```

### 4. Validación final

El resultado esperado es:

* El pod del `Job` `report-once` pasa de `Pending` a `Running` y después a `Completed`.
* El `Job` `report-once` aparece con `COMPLETIONS 1/1`.
* El `CronJob` puede ejecutar sus jobs periódicos.
* Los eventos de `FailedScheduling` dejan de repetirse para nuevos pods.

## Limpieza

```bash
kubectl delete ns cka-scheduling-lab
```

Si has añadido etiquetas a nodos, elimínalas al final:

```bash
kubectl label node <NODO> cka.formacion.io/pool-
```
