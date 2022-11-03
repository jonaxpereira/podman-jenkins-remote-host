# Utilizando Podman para ejecutar un servidor de Jenkins

Este proyecto tiene como objetivo demostrar el poder, simplicidad y gran compatibilidad que tiene Podman para ejecutar contenedores, en este caso, para levantar un servidor de jenkins que ejecuta scripts remotos por medio de un segundo contenedor.

## Prerequisitos

1. Instalar Podman
2. Instalar Podman Compose
3. Un editor de codigo (Editor utilizado: VS Code)
4. Sistema operativo a elección (S.O. utilizado: Windows 11)

## Creando el host remoto

Como mencionamos en un principio, haremos que Jenkins ejecute procesos remotos por medio de un segundo contenedor, al cual se conectará vía SSH.

Para el Host Remoto, utilizaremos un archivo Containerfile dentro de una carpeta llamada **centos** para construir la imagen de Podman del host remoto, el cual tendrá la siguiente configuración:

```Docker
FROM centos:7

# Install ssh server
RUN yum -y install openssh-server

# Setup user
RUN useradd remote_user && echo "qwerty" | passwd remote_user --stdin

# create .ssh folder for the remote_user user
RUN mkdir /home/remote_user/.ssh && chmod 700 /home/remote_user/.ssh

# Setup ssh key
COPY remote-key.pub /home/remote_user/.ssh/authorized_keys

# Setup permissions
RUN chown remote_user:remote_user -R /home/remote_user && \
    chmod 600 /home/remote_user/.ssh/authorized_keys

# "sshd: no hostkeys available" [fix error]
RUN /usr/sbin/sshd-keygen > /dev/null 2>&1

CMD /usr/sbin/sshd -D
```

Trabajaremos con una imagen base de centos 7, en la cual instalaremos el paquete ssh-server para ejecutar un servidor de ssh. Coniguraremos un usuario llamado **remote_user** y crearemos su propia carpeta en el directorio **home**.

En la línea bajo el comentario **Setup ssh key**, se copia un archivo de clave pública, el cual debe ser generado previo a la creación de la imagen de Podman. Para esto, dentro de la misma carpeta **centos** ejecutaremos el siguiente comando:

```
$ ssh-keygen -m PEM -t rsa -b 4096 -f remote-key
```

* ssh-keygen : comando para generar par de claves cifradas
* -m : formato de la clave
* -t : tipo de encriptación
* -b : cantidad de bits utilizados para encriptar
* -f : archivo de salida del resultado del comando

El comando anterior nos generará dos archivos, un archivo sin extensión que representa la clave privada ``remote-key`` y un archivo con extension ``.pub`` que representa la clave publica ``remote-key.pub``, este último es el archivo que podman copiará dentro de la imagen.

## Configurando compose.yaml

Para el ejercicio, utilizaremos podman-compose que es similar a utilizar docker-compose. Para ello, crearemos un archivo en formato ``.yaml`` como manifiesto de la configuración para desplegar tanto el contenedor de Jenkins como el contenedor del host-remoto