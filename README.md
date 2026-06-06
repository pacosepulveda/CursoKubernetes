# CKA AWS Labs

Repositorio de laboratorios para un curso práctico de **Kubernetes** orientado a la certificación **Certified Kubernetes Administrator (CKA)**.

El objetivo del repositorio es proporcionar una estructura clara para desplegar entornos de laboratorio en AWS, instalar Kubernetes con `kubeadm` y realizar prácticas progresivas sobre administración, operación y resolución de problemas en clústeres Kubernetes.

---

## Objetivo del curso

Este repositorio está pensado para alumnos que necesitan practicar la administración real de Kubernetes sobre máquinas Linux.

Durante el curso se trabaja con un clúster Kubernetes de tres nodos:

- 1 nodo **control-plane / master**
- 2 nodos **worker**
- Ubuntu Server 22.04
- `containerd` como runtime de contenedores
- `kubeadm`, `kubelet` y `kubectl`
- Calico como plugin de red CNI
- Laboratorios prácticos de despliegue, servicios, almacenamiento, scheduling, seguridad, Helm, actualización y troubleshooting

---

## Arquitectura del laboratorio

Cada entorno de laboratorio utiliza tres instancias EC2 en AWS.

```text
AWS Account / Entorno de alumno
└── VPC
    └── Subnet pública
        ├── Master   - 10.0.0.100
        ├── Worker1  - 10.0.0.101
        └── Worker2  - 10.0.0.102
```

Las IPs privadas se mantienen fijas para facilitar las instrucciones del curso y evitar que los comandos cambien entre alumnos.

---

## Instalación inicial de Docker

Antes de instalar Kubernetes, se puede instalar Docker en el nodo master para explicar conceptos básicos de contenedores:

- imágenes
- contenedores
- puertos publicados
- logs
- ejecución interactiva
- construcción de imágenes

La instalación de Docker está documentada en:

```text
Ejercicios/Instalar_Docker.md
```

Docker se utiliza como herramienta didáctica inicial. Kubernetes se instalará posteriormente usando `containerd` como runtime de contenedores.

---

## Instalación de Kubernetes

La instalación de Kubernetes con `kubeadm` está documentada en:

```text
Ejercicios/Instalar_Kubernetes.md
```

La guía incluye:

- limpieza de instalaciones fallidas previas
- desactivación de swap
- configuración de módulos del kernel
- configuración de parámetros `sysctl`
- instalación y configuración de `containerd`
- instalación de Kubernetes desde repositorios oficiales
- configuración de hostnames
- inicialización del master
- instalación de Calico
- unión de los nodos worker

---

## Versión de Kubernetes

El entorno está pensado para instalar una versión anterior a la última disponible, con el objetivo de realizar posteriormente una práctica de actualización del clúster.

Ejemplo:

```text
Versión inicial: Kubernetes 1.35.x
Actualización:   Kubernetes 1.36.x
```

Esto permite practicar una actualización realista sin tener que pasar por múltiples versiones intermedias.

---

## Advertencias importantes

Este repositorio está orientado a entornos de laboratorio.

No debe utilizarse tal cual en producción.

Aspectos a revisar antes de usarlo fuera de un entorno formativo:

- reglas del Security Group
- exposición de SSH
- exposición de NodePorts
- uso de Elastic IPs
- gestión de claves SSH
- permisos IAM
- limpieza de recursos al finalizar
- costes asociados a instancias, discos y direcciones IP públicas

---

## Requisitos previos

Para seguir las prácticas se recomienda tener conocimientos básicos de:

- Linux
- línea de comandos
- SSH
- redes TCP/IP
- contenedores
- YAML
- conceptos básicos de cloud computing

---

## Comandos útiles

Comprobar nodos:

```bash
kubectl get nodes
```

Comprobar pods de todos los namespaces:

```bash
kubectl get pods -A
```

Ver información del clúster:

```bash
kubectl cluster-info
```

Ver eventos:

```bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Generar un nuevo comando de unión para workers:

```bash
kubeadm token create --print-join-command
```

---

## Limpieza del clúster

Si una instalación falla y se quiere empezar de nuevo, se puede usar `kubeadm reset` y eliminar los directorios relacionados con Kubernetes y CNI.

Consulta la guía de instalación para ver el procedimiento completo.

---

## Licencia

Este material se proporciona con fines formativos.

Antes de reutilizarlo o publicarlo en otros cursos, revisa la licencia del repositorio y adapta los nombres, scripts, cuentas y configuraciones a tu entorno.
