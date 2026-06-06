# Instalación de Docker en Ubuntu

Esta guía muestra cómo instalar Docker en un nodo Ubuntu usando los paquetes disponibles en los repositorios de la distribución. Está pensada como paso previo a una introducción práctica a contenedores antes de trabajar con Kubernetes.

> **Nota:** en este laboratorio Docker se utiliza con fines didácticos para practicar conceptos básicos de contenedores: imágenes, contenedores, puertos, logs, ejecución interactiva y construcción de imágenes. Kubernetes se instalará posteriormente usando `containerd` como runtime de contenedores.

## 1. Actualizar los repositorios del sistema

Antes de instalar paquetes nuevos, actualizamos el índice de paquetes de Ubuntu:

```bash
sudo apt-get update
```

Este comando descarga la información actualizada de los repositorios configurados en el sistema. No actualiza los paquetes instalados, solo refresca la lista de versiones disponibles.

## 2. Instalar Docker

Instalamos el paquete `docker.io`:

```bash
sudo apt-get install -y docker.io
```

El parámetro `-y` acepta automáticamente la instalación de los paquetes necesarios.

En Ubuntu, el paquete `docker.io` instala Docker Engine desde los repositorios de la distribución. Para un laboratorio formativo es una opción sencilla y suficiente, aunque en entornos de producción puede preferirse instalar Docker desde los repositorios oficiales de Docker.

## 3. Habilitar el servicio Docker

Activamos Docker para que arranque automáticamente al iniciar el sistema:

```bash
sudo systemctl enable docker
```

Con esto, si la máquina se reinicia, el servicio Docker se iniciará de forma automática.

## 4. Iniciar el servicio Docker

Arrancamos Docker en la sesión actual:

```bash
sudo systemctl start docker
```

Podemos comprobar el estado del servicio con:

```bash
sudo systemctl status docker
```

Si el servicio está funcionando correctamente, debería aparecer como `active (running)`.

## 5. Permitir que el usuario `ubuntu` use Docker sin `sudo`

Por defecto, Docker requiere privilegios de administrador. Para evitar ejecutar todos los comandos Docker con `sudo`, añadimos el usuario `ubuntu` al grupo `docker`:

```bash
sudo usermod -aG docker ubuntu
```

El parámetro `-aG` añade el usuario al grupo indicado sin eliminarlo de otros grupos a los que ya pertenezca.

> **Importante:** pertenecer al grupo `docker` concede privilegios elevados sobre el sistema, ya que un usuario con acceso a Docker puede lanzar contenedores con acceso al host. En este laboratorio se hace por comodidad, pero en producción debe evaluarse cuidadosamente.

## 6. Aplicar el cambio de grupo en la sesión actual

Normalmente, después de añadir el usuario al grupo `docker`, habría que cerrar la sesión SSH y volver a entrar. Para evitarlo, podemos cambiar la shell actual al grupo `docker` ejecutando:

```bash
newgrp docker
```

Después de este comando, la sesión actual ya debería poder ejecutar comandos Docker sin usar `sudo`.

## 7. Comprobar la instalación

Ejecutamos:

```bash
docker ps
```

Si Docker está correctamente instalado y el usuario tiene permisos, el comando debería mostrar una tabla similar a esta:

```text
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

Aunque no haya contenedores en ejecución, el comando no debería devolver un error de permisos.

También podemos ejecutar un contenedor de prueba:

```bash
docker run hello-world
```

Este comando descarga una imagen de prueba y ejecuta un contenedor que muestra un mensaje de confirmación.

## Resumen de comandos

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu
newgrp docker
```

Comprobación:

```bash
docker ps
docker run hello-world
```
