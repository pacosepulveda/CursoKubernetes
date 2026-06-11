# Ejercicios Módulo 11 - Seguridad: RBAC y NetworkPolicies

> Entorno previsto: clúster Kubernetes instalado con `kubeadm` sobre instancias EC2 Ubuntu 22.04, runtime `containerd` y CNI Calico ya instalado.

---

## 1. Comprobaciones iniciales

Comprueba que el clúster está operativo:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

Comprueba que Calico está instalado. Dependiendo de cómo se haya instalado, los pods estarán en `kube-system`, `calico-system` o ambos:

```bash
kubectl get pods -A | grep -i calico
```

También podemos comprobar que el API de NetworkPolicy está disponible:

```bash
kubectl api-resources | grep -i networkpolicies
```

---

## 2. RBAC

RBAC regula qué puede hacer una identidad autenticada dentro de Kubernetes. Los objetos básicos son:

- `Role`: permisos dentro de un namespace.
- `ClusterRole`: permisos de ámbito clúster, o reutilizables desde un `RoleBinding`.
- `RoleBinding`: asigna un `Role` o `ClusterRole` a un usuario, grupo o ServiceAccount dentro de un namespace.
- `ClusterRoleBinding`: asigna permisos con ámbito de clúster.

En este laboratorio no vamos a crear usuarios IAM ni a tocar `aws-auth`, porque eso solo aplica a EKS. Usaremos una `ServiceAccount` llamada `rbac-student` para representar una identidad limitada.

### 2.1 Crear el namespace, la aplicación, la ServiceAccount y los permisos

Aplica el manifiesto:

```bash
kubectl apply -f modulo11-rbac.yaml
```

Comprueba los recursos:

```bash
kubectl get all -n rbac-test
kubectl get sa -n rbac-test
kubectl get role,rolebinding -n rbac-test
```

El manifiesto crea:

- Namespace `rbac-test`.
- Deployment `nginx`.
- ServiceAccount `rbac-student`.
- Role `workload-reader`, con permisos `get`, `list` y `watch` sobre pods y deployments.
- RoleBinding que une la ServiceAccount con el Role.

### 2.2 Comprobar permisos con `kubectl auth can-i`

Probamos qué puede hacer la ServiceAccount:

```bash
kubectl auth can-i list pods \
  -n rbac-test \
  --as=system:serviceaccount:rbac-test:rbac-student
```

Resultado esperado:

```text
yes
```

Puede listar deployments en su namespace:

```bash
kubectl auth can-i list deployments.apps \
  -n rbac-test \
  --as=system:serviceaccount:rbac-test:rbac-student
```

Resultado esperado:

```text
yes
```

No puede listar pods en otro namespace:

```bash
kubectl auth can-i list pods \
  -n kube-system \
  --as=system:serviceaccount:rbac-test:rbac-student
```

Resultado esperado:

```text
no
```

No puede borrar pods en su propio namespace, porque solo le dimos `get`, `list` y `watch`:

```bash
kubectl auth can-i delete pods \
  -n rbac-test \
  --as=system:serviceaccount:rbac-test:rbac-student
```

Resultado esperado:

```text
no
```

### 2.3 Ejecutar comandos simulando esa identidad

El usuario administrador del clúster puede impersonar otras identidades. Esto nos permite probar RBAC sin crear kubeconfigs nuevos.

Listar pods como `rbac-student`:

```bash
kubectl get pods -n rbac-test \
  --as=system:serviceaccount:rbac-test:rbac-student
```

Debe funcionar.

Intentar listar pods en `kube-system`:

```bash
kubectl get pods -n kube-system \
  --as=system:serviceaccount:rbac-test:rbac-student
```

Debe fallar con un error similar a:

```text
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:rbac-test:rbac-student" cannot list resource "pods" in API group "" in the namespace "kube-system"
```

### 2.4 Probar acceso denegado a Secrets

Creamos un Secret de prueba:

```bash
kubectl apply -f modulo11-rbac-deny-secret.yaml
```

El administrador puede verlo:

```bash
kubectl get secret demo-secret -n rbac-test
```

Pero `rbac-student` no debe poder leer Secrets:

```bash
kubectl get secret demo-secret -n rbac-test \
  --as=system:serviceaccount:rbac-test:rbac-student
```

Resultado esperado:

```text
Error from server (Forbidden): secrets "demo-secret" is forbidden
```

### 2.5 Crear un token temporal de la ServiceAccount, opcional

En Kubernetes moderno, los tokens de ServiceAccount ya no se crean automáticamente como Secrets permanentes. Podemos crear un token temporal así:

```bash
kubectl -n rbac-test create token rbac-student --duration=1h
```

Este token podría usarse para construir un kubeconfig limitado, aunque para el ejercicio CKA suele ser suficiente validar permisos con `kubectl auth can-i` y `--as`.

---

## 3. NetworkPolicies con Calico

Por defecto, si no hay políticas de red que seleccionen un pod, Kubernetes permite la comunicación entre pods. Las `NetworkPolicy` permiten restringir tráfico de entrada y/o salida a nivel L3/L4. Para que se apliquen, el CNI debe soportarlas. En este laboratorio usamos Calico.

Vamos a crear tres componentes en el namespace `netpol-demo`:

- `client`: pod desde el que haremos pruebas.
- `frontend`: servicio HTTP interno.
- `backend`: servicio HTTP interno.

El flujo final deseado será:

```text
client -> frontend -> backend
```

Pero no permitiremos acceso directo de `client` a `backend`.

### 3.1 Crear workloads de prueba

```bash
kubectl apply -f modulo11-netpol-workloads.yaml
kubectl -n netpol-demo get pods,svc -o wide
```

Espera a que todo esté en `Running`:

```bash
kubectl -n netpol-demo wait --for=condition=Ready pod/client --timeout=120s
kubectl -n netpol-demo rollout status deployment/frontend
kubectl -n netpol-demo rollout status deployment/backend
```

### 3.2 Probar comunicación antes de aplicar políticas

Desde el pod `client`, prueba el acceso a `frontend`:

```bash
kubectl -n netpol-demo exec client -- wget -qO- http://frontend
```

Resultado esperado:

```text
frontend OK
```

Prueba acceso directo a `backend`:

```bash
kubectl -n netpol-demo exec client -- wget -qO- http://backend
```

Resultado esperado:

```text
backend OK
```

Mientras no existan NetworkPolicies que seleccionen estos pods, el tráfico está permitido.

### 3.3 Aplicar denegación por defecto de entrada

Aplicamos una política que selecciona todos los pods del namespace y bloquea todo el tráfico entrante que no esté permitido explícitamente:

```bash
kubectl apply -f modulo11-netpol-default-deny.yaml
kubectl -n netpol-demo get networkpolicy
```

Ahora `client` ya no debería poder acceder a `frontend` ni a `backend`:

```bash
kubectl -n netpol-demo exec client -- wget -T 3 -qO- http://frontend || echo "bloqueado"
kubectl -n netpol-demo exec client -- wget -T 3 -qO- http://backend || echo "bloqueado"
```

Resultado esperado:

```text
bloqueado
bloqueado
```

### 3.4 Permitir tráfico de `client` a `frontend`

```bash
kubectl apply -f modulo11-netpol-allow-client-frontend.yaml
```

Ahora `client` sí debe poder acceder a `frontend`:

```bash
kubectl -n netpol-demo exec client -- wget -qO- http://frontend
```

Resultado esperado:

```text
frontend OK
```

Pero sigue sin poder acceder directamente a `backend`:

```bash
kubectl -n netpol-demo exec client -- wget -T 3 -qO- http://backend || echo "bloqueado"
```

Resultado esperado:

```text
bloqueado
```

### 3.5 Permitir tráfico de `frontend` a `backend`

```bash
kubectl apply -f modulo11-netpol-allow-frontend-backend.yaml
```

Ahora probamos desde el pod de `frontend` hacia el servicio `backend`:

```bash
FRONTEND_POD=$(kubectl -n netpol-demo get pod -l app=frontend -o jsonpath='{.items[0].metadata.name}')
kubectl -n netpol-demo exec "$FRONTEND_POD" -- wget -qO- http://backend
```

Resultado esperado:

```text
backend OK
```

Y comprobamos que `client` sigue sin acceder directamente a `backend`:

```bash
kubectl -n netpol-demo exec client -- wget -T 3 -qO- http://backend || echo "bloqueado"
```

Resultado esperado:

```text
bloqueado
```

### 3.6 Inspeccionar las políticas

```bash
kubectl -n netpol-demo describe networkpolicy default-deny-ingress
kubectl -n netpol-demo describe networkpolicy allow-client-to-frontend
kubectl -n netpol-demo describe networkpolicy allow-frontend-to-backend
```

También podemos revisar los endpoints para confirmar que los Services apuntan a los pods esperados:

```bash
kubectl -n netpol-demo get endpoints
```

### 3.7 Nota sobre DNS y políticas de egress

En este ejercicio solo hemos restringido `Ingress`. Si también creamos políticas de `Egress`, hay que permitir explícitamente DNS hacia CoreDNS; si no, nombres como `frontend` o `backend` dejarán de resolverse.

El archivo `modulo11-netpol-allow-dns-egress-opcional.yaml` muestra un ejemplo de permiso DNS, pero no es necesario aplicarlo en el flujo principal del ejercicio.

---

## 4. Limpieza

Eliminar recursos de RBAC:

```bash
kubectl delete -f modulo11-rbac-deny-secret.yaml --ignore-not-found
kubectl delete -f modulo11-rbac.yaml --ignore-not-found
```

Eliminar recursos de NetworkPolicies:

```bash
kubectl delete -f modulo11-netpol-allow-frontend-backend.yaml --ignore-not-found
kubectl delete -f modulo11-netpol-allow-client-frontend.yaml --ignore-not-found
kubectl delete -f modulo11-netpol-default-deny.yaml --ignore-not-found
kubectl delete -f modulo11-netpol-workloads.yaml --ignore-not-found
```

Comprobar que no quedan namespaces del ejercicio:

```bash
kubectl get ns | grep -E 'rbac-test|netpol-demo' || true
```

---

