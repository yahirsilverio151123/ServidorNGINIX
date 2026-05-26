# Proyecto: Implementación de NGINX y PHP desde Código Fuente

---

# Carátula

| Datos                           | Información                                                               |
| ------------------------------- | ------------------------------------------------------------------------- |
| **Institución**                 | Tecnológico de Estudios Superiores del Oriente del Estado de México       |
| **Asignatura**                  | Taller de Sistemas Operativos                                             |
| **Docente**                     | Gustavo Moisés Romero González                                            |
| **Tema del Proyecto**           | Instalación y configuración de NGINX y PHP compilados desde código fuente |
| **Sistema Operativo Utilizado** | AlmaLinux 10.1                                                            |
| **Fecha de entrega**            | Mayo 2026                                                                 |

## Integrantes

* PÉREZ VILLA YAHIR SILVERIO
* MARTÍNEZ MILLÁN ALEXANDER
* ALCÁNTARA GONZÁLEZ RUTH DAFNE

---

# Objetivo General

Desarrollar un entorno servidor web funcional utilizando NGINX 1.31.x y PHP 8.4.x compilados manualmente desde código fuente en AlmaLinux 10.1, estableciendo la comunicación mediante FastCGI con sockets UNIX y administrando ambos servicios a través de SystemD.

---

# Objetivos Específicos

1. Instalar y preparar AlmaLinux 10.1 en una máquina virtual.
2. Compilar e instalar NGINX 1.31.x usando el prefijo `/srv/nginx`.
3. Configurar usuarios y grupos del sistema para ejecutar los servicios con permisos restringidos.
4. Compilar PHP 8.4.x habilitando PHP-FPM, soporte gráfico e internacionalización.
5. Configurar la comunicación entre NGINX y PHP-FPM mediante socket UNIX.
6. Registrar los servicios en SystemD para inicio automático del sistema.
7. Verificar el correcto funcionamiento del servidor ejecutando archivos PHP desde el navegador.

---

# Desarrollo del Proyecto

## 1. Instalación del entorno base

Se creó una máquina virtual en Oracle VirtualBox con AlmaLinux 10.1 utilizando las siguientes características:

* Memoria RAM: 4 GB
* Procesador: 2 núcleos
* Disco duro virtual: 20 GB dinámico
* Adaptador de red NAT

Después de instalar el sistema operativo, se actualizó el sistema con:

```bash
dnf update -y
```

Posteriormente se instalaron las herramientas y dependencias necesarias para compilar NGINX y PHP:

```bash
dnf install -y gcc gcc-c++ make cmake wget tar git \
openssl-devel zlib-devel libcurl-devel sqlite-devel \
libpng-devel freetype-devel oniguruma-devel \
bison nano epel-release
```

---

## 2. Creación de usuarios y estructura de directorios

Para ejecutar los servicios de forma segura se crearon usuarios del sistema sin acceso interactivo:

```bash
groupadd -r nginx

useradd -r -g nginx -s /sbin/nologin -d /srv/nginx nginx

useradd -r -g nginx -s /sbin/nologin php
```

Después se generó el directorio principal donde se instalarían ambos servicios:

```bash
mkdir -p /srv/nginx

chown -R nginx:nginx /srv/nginx
```

---

## 3. Instalación de NGINX 1.31.x desde código fuente

Se descargó la versión oficial de NGINX:

```bash
cd /tmp

wget https://nginx.org/download/nginx-1.31.1.tar.gz

tar -xvzf nginx-1.31.1.tar.gz

cd nginx-1.31.1
```

La compilación se realizó utilizando módulos importantes como SSL y HTTP/2:

```bash
./configure \
--prefix=/srv/nginx \
--user=nginx \
--group=nginx \
--with-http_ssl_module \
--with-http_v2_module \
--with-http_gzip_static_module \
--with-stream \
--with-pcre
```

Posteriormente se compiló e instaló:

```bash
make && make install
```

Para confirmar la instalación:

```bash
/srv/nginx/sbin/nginx -v
```

Resultado esperado:

```bash
nginx version: nginx/1.31.1
```

---

## 4. Instalación de PHP 8.4.x desde código fuente

Se descargó el código fuente de PHP:

```bash
cd /tmp

wget https://www.php.net/distributions/php-8.4.7.tar.gz

tar -xvzf php-8.4.7.tar.gz

cd php-8.4.7
```

La configuración se realizó habilitando PHP-FPM, procesamiento de imágenes e internacionalización:

```bash
./configure \
--prefix=/srv/nginx \
--enable-fpm \
--with-fpm-user=php \
--with-fpm-group=nginx \
--with-openssl \
--with-zlib \
--with-curl \
--enable-gd \
--with-freetype \
--with-jpeg \
--with-webp \
--enable-intl \
--enable-calendar \
--enable-mbstring \
--with-sqlite3
```

Después se compiló e instaló:

```bash
make && make install
```

---

## 5. Configuración de PHP-FPM

Se copiaron los archivos base de configuración:

```bash
cp /srv/nginx/etc/php-fpm.conf.default \
/srv/nginx/etc/php-fpm.conf

cp /srv/nginx/etc/php-fpm.d/www.conf.default \
/srv/nginx/etc/php-fpm.d/www.conf
```

Dentro del archivo:

```bash
/srv/nginx/etc/php-fpm.d/www.conf
```

Se modificaron los siguientes parámetros:

```ini
user = php
group = nginx

listen = /tmp/php84.sock

listen.owner = nginx
listen.group = nginx
listen.mode = 0660
```

Con esta configuración PHP-FPM utilizaría un socket UNIX para comunicarse con NGINX.

---

## 6. Configuración de NGINX y FastCGI

Se modificó el archivo principal:

```bash
/srv/nginx/conf/nginx.conf
```

Configuración utilizada:

```nginx
user nginx nginx;

worker_processes auto;

events {
    worker_connections 1024;
}

http {

    include mime.types;

    default_type application/octet-stream;

    sendfile on;

    keepalive_timeout 65;

    server {

        listen 80;

        server_name localhost;

        root /srv/nginx/html;

        index index.php index.html;

        location / {
            try_files $uri $uri/ =404;
        }

        location ~ \.php$ {

            fastcgi_pass unix:/tmp/php84.sock;

            fastcgi_index index.php;

            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;

            include fastcgi_params;
        }
    }
}
```

Esta configuración permitió que NGINX enviara las peticiones PHP hacia PHP-FPM utilizando FastCGI.

---

## 7. Registro de servicios en SystemD

### Servicio NGINX

Archivo:

```bash
/etc/systemd/system/nginx.service
```

Contenido:

```ini
[Unit]
Description=NGINX Web Server
After=network.target

[Service]
Type=forking
PIDFile=/srv/nginx/logs/nginx.pid
ExecStartPre=/srv/nginx/sbin/nginx -t
ExecStart=/srv/nginx/sbin/nginx
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID

[Install]
WantedBy=multi-user.target
```

---

### Servicio PHP-FPM

Archivo:

```bash
/etc/systemd/system/php-fpm8.4.service
```

Contenido:

```ini
[Unit]
Description=PHP 8.4 FastCGI Process Manager
After=network.target

[Service]
Type=simple
ExecStart=/srv/nginx/sbin/php-fpm \
--nodaemonize \
--fpm-config /srv/nginx/etc/php-fpm.conf

ExecReload=/bin/kill -USR2 $MAINPID

[Install]
WantedBy=multi-user.target
```

Se habilitaron ambos servicios:

```bash
systemctl daemon-reload

systemctl enable nginx.service

systemctl enable php-fpm8.4.service
```

---

## 8. Prueba de funcionamiento

Para comprobar el funcionamiento del entorno web se creó el siguiente archivo:

```bash
echo '<?php phpinfo(); ?>' > /srv/nginx/html/phpinfo.php
```

Se inició el servicio:

```bash
systemctl start nginx

systemctl start php-fpm8.4.service
```

Desde el navegador se accedió a:

```text
http://127.0.0.1:8080/phpinfo.php
```

La página mostró correctamente la información de PHP, confirmando que la comunicación entre NGINX y PHP-FPM funcionaba correctamente mediante FastCGI y sockets UNIX.

---

# Conclusiones

La implementación de NGINX y PHP desde código fuente permitió comprender de manera práctica el funcionamiento interno de un entorno servidor Linux.

Durante el desarrollo del proyecto se realizaron tareas relacionadas con compilación manual, configuración de servicios, administración de permisos y uso de SystemD para automatizar procesos de arranque.

También se aprendió el funcionamiento de FastCGI y la comunicación mediante sockets UNIX entre NGINX y PHP-FPM, logrando un entorno estable y completamente funcional para ejecutar aplicaciones PHP.

Finalmente, el proyecto permitió reforzar conocimientos sobre administración de sistemas Linux y despliegue de servicios web.

---

# Bibliografía

nginx. (2026). *NGINX download and installation guide*. Recuperado de [https://nginx.org/en/download.html](https://nginx.org/en/download.html)

PHP Group. (2026). *PHP source code downloads*. Recuperado de [https://www.php.net/downloads.php](https://www.php.net/downloads.php)

AlmaLinux OS Foundation. (2026). *AlmaLinux documentation*. Recuperado de [https://almalinux.org/](https://almalinux.org/)

Red Hat, Inc. (2026). *Systemd unit configuration*. Recuperado de [https://docs.redhat.com/](https://docs.redhat.com/)

Oracle Corporation. (2026). *Oracle VM VirtualBox documentation*. Recuperado de [https://www.virtualbox.org/manual/](https://www.virtualbox.org/manual/)

nginx. (2026). *NGINX FastCGI configuration guide*. Recuperado de [https://nginx.org/en/docs/](https://nginx.org/en/docs/)
