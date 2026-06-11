# Ejercicio — Troubleshooting en Kubernetes: CrashLoopBackOff, OOMKilled y dependencias de configuración

## Contexto del laboratorio

Trabajas como administrador de una plataforma Kubernetes desplegada con `kubeadm` sobre máquinas EC2 en AWS.

El clúster tiene la siguiente arquitectura:

- 1 nodo control plane / master.
- 2 nodos worker.
- Instancias EC2 `t3.medium`.
- CNI instalado: Calico.
- Acceso al clúster mediante `kubectl`.

El equipo de desarrollo ha desplegado varias aplicaciones de prueba en el namespace `lab-troubleshooting`, pero ninguna funciona correctamente. Tu tarea consiste en diagnosticar los problemas, identificar la causa raíz y aplicar la corrección adecuada.

El objetivo del ejercicio no es “arreglar a ciegas”, sino seguir una metodología ordenada de troubleshooting.

---

## Objetivos

Al finalizar este ejercicio deberías ser capaz de:

- Identificar distintos estados de fallo en Pods.
- Diferenciar un `CrashLoopBackOff` de un `CreateContainerConfigError`.
- Usar `kubectl get`, `kubectl describe` y `kubectl logs` para diagnosticar problemas.
- Interpretar eventos de Kubernetes.
- Detectar reinicios de contenedores.
- Identificar un fallo por salida de proceso, un fallo por límite de memoria y un fallo por dependencia de configuración.
- Aplicar una corrección y verificar que el recurso queda estable.

---

## Archivos proporcionados

Para realizar el ejercicio se proporcionan los siguientes archivos:

```text
ns.yaml
scenario1-bug-deploy.yaml
scenario2-oom-deploy.yaml
scenario3-config-dependency-pod.yaml
```

El instructor puede proporcionar otros archivos de corrección cuando corresponda.

---

## Preparación del entorno

Antes de empezar, comprueba que puedes acceder al clúster:

```bash
kubectl get nodes -o wide
```

Verifica que los nodos aparecen en estado `Ready`.

A continuación, crea el namespace del laboratorio:

```bash
kubectl apply -f ns.yaml
```

Comprueba que se ha creado correctamente:

```bash
kubectl get ns lab-troubleshooting
```

---

## Metodología recomendada

Para cada escenario debes seguir siempre una secuencia similar:

1. Desplegar el recurso problemático.
2. Observar el estado del Pod.
3. Identificar el nombre exacto del Pod afectado.
4. Revisar la descripción del Pod.
5. Revisar los eventos.
6. Revisar logs cuando el contenedor haya llegado a arrancar.
7. Formular una hipótesis de causa raíz.
8. Aplicar una corrección.
9. Verificar que el recurso queda estable.

No saltes directamente a aplicar correcciones. El objetivo es aprender a diagnosticar.

---

# Escenario 1 — Aplicación que entra en CrashLoopBackOff

## Despliegue

Aplica el manifiesto problemático:

```bash
kubectl apply -f scenario1-bug-deploy.yaml
```

## Tareas

Debes responder a las siguientes preguntas:

1. ¿Qué recurso se ha creado?
2. ¿Qué Pod se ha generado a partir del Deployment?
3. ¿En qué estado queda el Pod?
4. ¿Cuántos reinicios acumula?
5. ¿El contenedor llega a arrancar?
6. ¿Cuál es el motivo de la terminación del contenedor?
7. ¿Qué información aportan los logs?
8. ¿Qué diferencia hay entre consultar los logs actuales y los logs del intento anterior?
9. ¿Cuál crees que es la causa raíz?
10. ¿Qué cambio sería necesario para que el contenedor permanezca en ejecución?

## Resultado esperado

El escenario debe quedar diagnosticado como un fallo de aplicación o comando de arranque, no como un problema de red, scheduling, permisos o imagen inexistente.

---

# Escenario 2 — Aplicación que falla por consumo de memoria

## Despliegue

Aplica el manifiesto problemático:

```bash
kubectl apply -f scenario2-oom-deploy.yaml
```

## Tareas

Debes responder a las siguientes preguntas:

1. ¿En qué estado queda el Pod?
2. ¿El patrón de fallo es igual o diferente al del escenario 1?
3. ¿Qué valor aparece en la sección `Last State` del contenedor?
4. ¿Qué significa `OOMKilled`?
5. ¿Qué límite de memoria tiene configurado el contenedor?
6. ¿Cuánta memoria intenta reservar el proceso?
7. ¿La aplicación falla por un error lógico o por una restricción de recursos?
8. ¿Qué opciones tendrías para corregir el problema?
9. ¿Es mejor aumentar el límite, reducir el consumo o ambas cosas?
10. ¿Cómo verificarías que el Pod queda estable después de corregirlo?

## Resultado esperado

El escenario debe quedar diagnosticado como un fallo causado por límite de memoria insuficiente.

---

# Escenario 3 — Error por dependencia de configuración

## Despliegue

Aplica el manifiesto problemático:

```bash
kubectl apply -f scenario3-config-dependency-pod.yaml
```

## Tareas

Debes responder a las siguientes preguntas:

1. ¿Qué estado muestra el Pod?
2. ¿El Pod entra en `CrashLoopBackOff`?
3. ¿El contenedor llega a arrancar?
4. ¿Tiene sentido consultar logs en este caso?
5. ¿Qué información aparece en los eventos del Pod?
6. ¿Qué dependencia externa necesita el Pod para poder arrancar?
7. ¿Qué objeto de Kubernetes falta?
8. ¿Qué clave concreta espera encontrar el contenedor?
9. ¿Por qué este error se detecta antes de arrancar el contenedor?
10. ¿Qué recurso habría que crear para que el Pod pueda iniciar?

## Resultado esperado

El escenario debe quedar diagnosticado como un error de configuración previo al arranque del contenedor.

---

## Entregable del alumno

Para completar el ejercicio, entrega un documento breve con la siguiente estructura:

```text
Escenario 1
- Estado observado:
- Comandos utilizados:
- Evidencias encontradas:
- Causa raíz:
- Corrección aplicada:
- Verificación final:

Escenario 2
- Estado observado:
- Comandos utilizados:
- Evidencias encontradas:
- Causa raíz:
- Corrección aplicada:
- Verificación final:

Escenario 3
- Estado observado:
- Comandos utilizados:
- Evidencias encontradas:
- Causa raíz:
- Corrección aplicada:
- Verificación final:
```

---

## Limpieza del laboratorio

Al finalizar el ejercicio, elimina el namespace completo:

```bash
kubectl delete ns lab-troubleshooting --ignore-not-found=true
```

Esto eliminará todos los recursos creados durante el laboratorio.
