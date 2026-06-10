# Ejercicio 01 - Solución explicada

## 1. Despliegue inicial del escenario

Aplicamos los manifiestos con errores intencionados:

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

Comprobamos el estado inicial:

```bash
kubectl get all -n m7-troubleshooting
kubectl get pv
kubectl get pvc -n m7-troubleshooting
```

Es normal que la aplicación no funcione todavía. El ejercicio está diseñado para que el alumno tenga que seguir una secuencia lógica de troubleshooting.

---

## 2. Problema 1: el PVC queda en estado `Pending`

Comprobamos el almacenamiento:

```bash
kubectl get pv
kubectl get pvc -n m7-troubleshooting
```

Resultado esperado aproximado:

```text
NAME                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS    ...
pv-reporting-local   1Gi        RWO            Retain           Available           manual-retain   ...
```

```text
NAME             STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
reporting-data   Pending                                      manual
```

El `PV` está `Available`, pero el `PVC` está `Pending`.

Investigamos el PVC:

```bash
kubectl describe pvc reporting-data -n m7-troubleshooting
```

El punto clave es comparar estos campos:

```yaml
# PV
storageClassName: manual-retain
```

```yaml
# PVC inicial
storageClassName: manual
```

El `PVC` solicita una clase de almacenamiento llamada `manual`, pero el `PV` disponible tiene `storageClassName: manual-retain`.

Aunque la capacidad y el modo de acceso coincidan, Kubernetes no enlaza el `PVC` con ese `PV` porque la clase de almacenamiento no coincide.

### Corrección

Aplicamos el PVC corregido:

```bash
kubectl apply -f yaml/fix/01-pvc-fixed.yaml
```

Comprobamos:

```bash
kubectl get pvc -n m7-troubleshooting
kubectl get pv
```

Ahora deberíamos ver el `PVC` en estado `Bound`:

```text
NAME             STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS
reporting-data   Bound    pv-reporting-local   1Gi        RWO            manual-retain
```

---

## 3. Problema 2: error al montar una clave del ConfigMap

Una vez corregido el `PVC`, Kubernetes ya puede intentar crear el Pod. Sin embargo, todavía falla.

Comprobamos el estado:

```bash
kubectl get pods -n m7-troubleshooting
```

Es posible ver el Pod en estado `ContainerCreating`, `Pending` o con eventos de fallo de montaje.

Obtenemos más detalle:

```bash
kubectl describe pod -n m7-troubleshooting -l app=reporting-web
```

En los eventos debería aparecer un mensaje parecido a este:

```text
MountVolume.SetUp failed for volume "config" : configmap references non-existent config key: application.properties
```

El `Deployment` monta un volumen basado en el `ConfigMap` `reporting-config`:

```yaml
volumes:
  - name: config
    configMap:
      name: reporting-config
      items:
        - key: application.properties
          path: application.properties
```

Pero el `ConfigMap` inicial contiene esta clave:

```yaml
data:
  app.properties: |
    app.name=Sistema de informes
```

El problema es que el Pod intenta montar una clave llamada `application.properties`, pero el `ConfigMap` contiene `app.properties`.

Cuando se usan `items` en un volumen de tipo `ConfigMap`, Kubernetes espera que todas las claves indicadas existan. Si una clave no existe, el volumen no puede montarse correctamente.

### Corrección

Aplicamos el `ConfigMap` corregido:

```bash
kubectl apply -f yaml/fix/02-configmap-fixed.yaml
```

Reiniciamos el Pod para forzar una nueva creación con la configuración corregida:

```bash
kubectl delete pod -n m7-troubleshooting -l app=reporting-web
```

Comprobamos otra vez:

```bash
kubectl get pods -n m7-troubleshooting
kubectl describe pod -n m7-troubleshooting -l app=reporting-web
```

---

## 4. Problema 3: clave inexistente en el Secret

Después de corregir el `ConfigMap`, el Pod avanza más en el proceso de arranque, pero todavía tiene otro problema.

Comprobamos los Pods:

```bash
kubectl get pods -n m7-troubleshooting
```

Puede aparecer un estado como:

```text
CreateContainerConfigError
```

Consultamos los detalles:

```bash
kubectl describe pod -n m7-troubleshooting -l app=reporting-web
```

En los eventos o en la descripción del contenedor debería aparecer un error relacionado con la clave `db-password`.

El `Deployment` define esta variable de entorno:

```yaml
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: reporting-db-secret
      key: db-password
```

Pero el `Secret` inicial contiene:

```yaml
stringData:
  db-user: report_user
  db_pass: supersecreto123
```

El nombre de la clave no coincide. El Pod busca `db-password`, pero el `Secret` tiene `db_pass`.

A diferencia de otros errores que se producen dentro de la aplicación, este error ocurre antes de arrancar el contenedor. Kubernetes no puede construir correctamente la configuración del contenedor porque una variable de entorno depende de una clave inexistente del `Secret`.

### Corrección

Aplicamos el `Secret` corregido:

```bash
kubectl apply -f yaml/fix/03-secret-fixed.yaml
```

Reiniciamos de nuevo el Pod:

```bash
kubectl delete pod -n m7-troubleshooting -l app=reporting-web
```

Comprobamos:

```bash
kubectl get pods -n m7-troubleshooting -o wide
```

Ahora el Pod debería llegar a `Running`:

```text
NAME                             READY   STATUS    RESTARTS   AGE
reporting-web-xxxxxxxxxx-yyyyy   1/1     Running   0          ...
```

---

## 5. Validación final

### 5.1 Validar PVC y PV

```bash
kubectl get pvc -n m7-troubleshooting
kubectl get pv
```

Resultado esperado:

```text
NAME             STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS
reporting-data   Bound    pv-reporting-local   1Gi        RWO            manual-retain
```

El `PVC` está enlazado al `PV` `pv-reporting-local`.

---

### 5.2 Validar el contenido generado por el initContainer

Obtenemos el nombre del Pod:

```bash
POD=$(kubectl get pod -n m7-troubleshooting -l app=reporting-web -o jsonpath='{.items[0].metadata.name}')
```

Leemos el archivo generado:

```bash
kubectl exec -n m7-troubleshooting "$POD" -c nginx -- cat /usr/share/nginx/html/index.html
```

Resultado esperado aproximado:

```html
<h1>Reporting Web</h1>
<p>Contenido generado por initContainer.</p>
<p>Usuario de BD: report_user</p>
```

El archivo está en el volumen persistente. Lo ha creado el `initContainer` y luego lo sirve el contenedor Nginx.

---

### 5.3 Validar el ConfigMap montado como volumen

```bash
kubectl exec -n m7-troubleshooting "$POD" -c nginx -- ls -l /etc/reporting
kubectl exec -n m7-troubleshooting "$POD" -c nginx -- cat /etc/reporting/application.properties
```

Resultado esperado:

```text
app.name=Sistema de informes
app.environment=lab
app.logLevel=INFO
```

Esto confirma que el `ConfigMap` está montado correctamente en el contenedor.

---

### 5.4 Validar el Service

Desde el Pod de debug:

```bash
kubectl exec -n m7-troubleshooting debug-shell -- curl -s http://reporting-web
```

Resultado esperado aproximado:

```html
<h1>Reporting Web</h1>
<p>Contenido generado por initContainer.</p>
<p>Usuario de BD: report_user</p>
```

La resolución DNS del Service funciona porque el Pod de debug y el Service están en el mismo namespace.

---

## 6. Explicación de los errores

### Error 1: `PVC Pending`

El `PVC` estaba en estado `Pending` porque su `storageClassName` no coincidía con el del `PV` disponible.

Para que un `PVC` se enlace a un `PV`, deben ser compatibles varios campos:

- Capacidad solicitada.
- `accessModes`.
- `volumeMode`.
- `storageClassName`.
- Selectores, si existen.

En este caso, la capacidad y el modo de acceso eran compatibles, pero la clase de almacenamiento no.

---

### Error 2: clave inexistente en ConfigMap

El `Deployment` intentaba montar una clave concreta del `ConfigMap`:

```yaml
- key: application.properties
  path: application.properties
```

Pero el `ConfigMap` tenía la clave `app.properties`.

Cuando se monta un `ConfigMap` completo sin `items`, Kubernetes monta todas las claves existentes como archivos. En cambio, cuando se usa `items`, Kubernetes solo monta las claves indicadas. Si una de esas claves no existe, el volumen no puede montarse.

---

### Error 3: clave inexistente en Secret

El contenedor principal intentaba crear la variable de entorno `DB_PASSWORD` desde esta clave:

```yaml
secretKeyRef:
  name: reporting-db-secret
  key: db-password
```

Pero el `Secret` tenía `db_pass`.

Como la clave no existe y no se ha configurado `optional: true`, Kubernetes no puede crear correctamente la configuración del contenedor y aparece un error como `CreateContainerConfigError`.

---

## 7. Persistencia de los datos

El contenido `index.html` se escribe en el volumen montado desde el `PVC`:

```yaml
volumeMounts:
  - name: data
    mountPath: /workdir
```

El contenedor Nginx monta el mismo volumen en:

```yaml
mountPath: /usr/share/nginx/html
```

Por eso el archivo generado por el `initContainer` queda disponible para Nginx.

Si se elimina el Pod y se crea de nuevo, el contenido no debería perderse porque está almacenado en el volumen persistente y no en el filesystem efímero del contenedor.

Prueba:

```bash
kubectl delete pod -n m7-troubleshooting -l app=reporting-web
kubectl wait --for=condition=Ready pod -n m7-troubleshooting -l app=reporting-web --timeout=120s
POD=$(kubectl get pod -n m7-troubleshooting -l app=reporting-web -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n m7-troubleshooting "$POD" -c nginx -- cat /usr/share/nginx/html/index.html
```

---

## 8. Nota importante sobre `hostPath`

Este ejercicio usa un `PersistentVolume` basado en `hostPath` para simplificar el laboratorio.

```yaml
hostPath:
  path: /tmp/k8s-m7-reporting
  type: DirectoryOrCreate
```

Esto es útil para aprender los conceptos de `PV` y `PVC`, pero no es una buena práctica para producción porque:

- Los datos quedan ligados a un nodo concreto.
- Si el Pod se planifica en otro nodo, podría usar otro directorio distinto con el mismo path.
- No hay replicación ni alta disponibilidad.
- La seguridad depende directamente del filesystem del nodo.
- Puede generar inconsistencias si se usa incorrectamente en clústeres multinodo.

En producción se usaría almacenamiento gestionado o compartido, por ejemplo:

- CSI Driver de AWS EBS.
- Azure Disk o Azure Files.
- Google Persistent Disk.
- NFS gestionado.
- Ceph, Longhorn, Portworx u otras soluciones de almacenamiento para Kubernetes.

---

## 9. Limpieza

Eliminamos el namespace:

```bash
kubectl delete namespace m7-troubleshooting
```

El `PV` no pertenece al namespace, por lo que se elimina aparte:

```bash
kubectl delete pv pv-reporting-local
```

Si quieres limpiar los datos creados en el nodo donde se haya ejecutado el Pod:

```bash
sudo rm -rf /tmp/k8s-m7-reporting
```

---

## 10. Resumen de comandos útiles usados en el troubleshooting

```bash
kubectl get all -n m7-troubleshooting
kubectl get pv
kubectl get pvc -n m7-troubleshooting
kubectl describe pvc reporting-data -n m7-troubleshooting
kubectl get pods -n m7-troubleshooting
kubectl describe pod -n m7-troubleshooting -l app=reporting-web
kubectl logs -n m7-troubleshooting -l app=reporting-web --all-containers
kubectl exec -n m7-troubleshooting "$POD" -c nginx -- cat /usr/share/nginx/html/index.html
kubectl exec -n m7-troubleshooting debug-shell -- curl -s http://reporting-web
```
