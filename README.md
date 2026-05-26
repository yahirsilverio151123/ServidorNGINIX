
# ServidorNGINIX# proyecto-moi
# Implementación de Servidor NGINX y PHP compilados desde Código Fuente

---

## Carátula

| | |
|---|---|
| **Escuela** | Tecnologico de Estudios Superiores del Oriente del Estado de Mexico |
| **Materia** | Taller de sistemas Operativos |
| **Profesor** | Gustavo Moises Romero Gonzalez |
| **Proyecto** | Implementación de NGINX 1.31.x y PHP 8.4.x compilados desde código fuente |
| **Sistema Operativo** | AlmaLinux 10.1 (Heliotrope Lion) |
| **Fecha** | Mayo 2026 |

### Integrantes

•	ALCANTARA GONZALEZ RUTH DAFNE

•	MARTINEZ MILLAN ALEXANDER

•	PEREZ VILLA YAHIR SILVERIO

•	CHAVARRIA MEZA MILDRED ALEXIA

---

## Objetivo General

Implementar un servidor web funcional compilando NGINX 1.31.x y PHP 8.4.x desde código fuente sobre AlmaLinux 10.1, configurando la comunicación entre ambos mediante FastCGI a través de un socket Unix, y registrando ambos servicios en systemd para que arranquen automáticamente con el sistema operativo.

---

## Objetivos Específicos

1. Instalar y configurar AlmaLinux 10.1 en una máquina virtual VirtualBox como entorno de servidor.
2. Compilar NGINX versión 1.31.1 desde código fuente con el prefijo de instalación en `/srv/nginx`.
3. Compilar PHP versión 8.4.7 desde código fuente habilitando soporte para FPM, procesamiento de imágenes, fechas e internacionalización.
4. Configurar PHP-FPM para comunicarse con NGINX mediante socket Unix en `/var/run/php84.sock`.
5. Crear y registrar unidades de servicio systemd para NGINX y PHP-FPM con arranque automático en `multi-user.target`.
6. Verificar el funcionamiento del stack mediante un script PHP visible en el navegador web.

---

## Desarrollo del Proyecto

### 1. Preparación del entorno

Se instaló AlmaLinux 10.1 en una máquina virtual Oracle VirtualBox con las siguientes características:

- RAM: 4096 MB
- CPU: 2 núcleos
- Disco: 20 GB (VDI dinámico)
- Red: NAT con reenvío de puertos (host 8080 → guest 80)

Una vez instalado el sistema, se actualizó y se instalaron las dependencias necesarias para compilar:

```bash
dnf update -y

<img width="1280" height="960" alt="image" src="https://github.com/user-attachments/assets/a7c1a789-77fc-42b8-ac04-8e932ba339cb" />
```bash
dnf install -y gcc gcc-c++ make cmake zlib-devel openssl-devel \
  libcurl-devel libpng-devel freetype-devel sqlite-devel \
  bison tar wget git oniguruma-devel epel-release nano
```

<img width="1600" height="899" alt="image" src="https://github.com/user-attachments/assets/64f89dc7-0696-437f-bc3f-ce70e5cfd818" />

<img width="1600" height="902" alt="image" src="https://github.com/user-attachments/assets/4274c87a-6170-4934-ba3a-15b2dba9cb0f" />

<img width="1600" height="899" alt="image" src="https://github.com/user-attachments/assets/7888d819-4104-4973-aedc-1acca03ae32a" />


### 2. Creación de usuarios y directorios del sistema

Se crearon los usuarios y grupos del sistema necesarios para ejecutar los servicios con privilegios mínimos:

```bash
groupadd -r nginx
useradd -r -g nginx -s /sbin/nologin -d /srv/nginx nginx
useradd -r -g nginx -s /sbin/nologin php.

<img width="1600" height="899" alt="image" src="https://github.com/user-attachments/assets/cabf8b92-6da2-4bdd-9700-ba2bcd54d022" />

```bash
mkdir -p /srv/nginx
chown -R nginx:nginx /srv/nginx
```


### 3. Compilación e instalación de NGINX 1.31.1

Se descargó el código fuente de NGINX versión 1.31.1 desde el sitio oficial y se compiló con las siguientes opciones:

```bash
cd /tmp
wget https://nginx.org/download/nginx-1.31.1.tar.gz
tar -xzvf nginx-1.31.1.tar.gz
cd nginx-1.31.1
```

<img width="1600" height="899" alt="image" src="https://github.com/user-attachments/assets/56d4a896-9250-4728-8795-ad82c76f1c76" />

```bash
./configure \
  --prefix=/srv/nginx \
  --user=nginx \
  --group=nginx \
  --with-http_ssl_module \
  --with-http_v2_module \
  --with-http_gzip_static_module \
  --with-pcre \
  --with-stream

make && make install
```

<img width="1600" height="899" alt="image" src="https://github.com/user-attachments/assets/f25b30a4-9e3e-40f9-aa0d-0188bb155080" />

```bash
make && make install
```
Se verificó la instalación con:

```bash
/srv/nginx/sbin/nginx -v
# nginx version: nginx/1.31.1
```


### 4. Compilación e instalación de PHP 8.4.7

Se descargó el código fuente de PHP versión 8.4.7 y se compiló habilitando FPM, soporte de imágenes (GD), internacionalización (intl) y calendario:

```bash
cd /tmp
wget https://www.php.net/distributions/php-8.4.7.tar.gz
tar -xzvf php-8.4.7.tar.gz
cd php-8.4.7
```


<img width="1600" height="899" alt="image" src="https://github.com/user-attachments/assets/d62eb242-3fe9-44fd-af58-00d5997ecc5c" />

```bash
./configure \
  --prefix=/srv/nginx \
  --with-fpm-user=php \
  --with-fpm-group=nginx \
  --enable-fpm \
  --with-openssl \
  --with-zlib \
  --with-curl \
  --enable-gd \
  --with-jpeg \
  --with-webp \
  --with-freetype \
  --enable-intl \
  --enable-mbstring \
  --enable-calendar \
  --with-sqlite3
```
### 5. Configuración de PHP-FPM con socket Unix

Se copiaron los archivos de configuración base y se modificó el pool `www` para usar socket Unix:

```bash
cp /srv/nginx/etc/php-fpm.conf.default /srv/nginx/etc/php-fpm.conf
cp /srv/nginx/etc/php-fpm.d/www.conf.default /srv/nginx/etc/php-fpm.d/www.conf
```

En `/srv/nginx/etc/php-fpm.d/www.conf` se configuraron los siguientes parámetros:

```ini
user = php
group = nginx
listen = /var/run/php84.sock
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
```

6. Configuración de NGINX con FastCGI
Se configuró NGINX para procesar archivos PHP mediante FastCGI a través del socket Unix. El archivo /srv/nginx/conf/nginx.confquedó de la siguiente manera:

user nginx nginx;
worker_processes 1;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile      on;
    keepalive_timeout 65;

    server {
        listen      80;
        server_name localhost;
        root        /srv/nginx/html;
        index       index.php index.html;

        location / {
            try_files $uri $uri/ =404;
        }

        location ~ \.php$ {
            fastcgi_pass  unix:/var/run/php84.sock;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include       fastcgi_params;
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
            root html;
        }
    }
}


h<img width="1286" height="821" alt="image" src="https://github.com/user-attachments/assets/d01c0e33-c209-4835-9d91-2ffa791ec199" />


### 7. Registro de servicios en systemd

**Servicio NGINX** — `/etc/systemd/system/nginx.service`:

```ini
[Unit]
Description=NGINX HTTP Server
After=network.target

[Service]
Type=forking
PIDFile=/srv/nginx/logs/nginx.pid
ExecStartPre=/srv/nginx/sbin/nginx -t
ExecStart=/srv/nginx/sbin/nginx
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```


<img width="1600" height="899" alt="image" src="https://github.com/user-attachments/assets/1a2a1553-7189-4f00-b227-db40c9b5780f" />



**Servicio PHP-FPM** — `/etc/systemd/system/php-fpm8.4.service`:

```ini
[Unit]
Description=PHP 8.4 FastCGI Process Manager
After=network.target

[Service]
Type=simple
PIDFile=/srv/nginx/var/run/php-fpm.pid
ExecStart=/srv/nginx/sbin/php-fpm --nodaemonize --fpm-config /srv/nginx/etc/php-fpm.conf
ExecReload=/bin/kill -s USR2 $MAINPID

[Install]
WantedBy=multi-user.target
```

### 8. Verificación del funcionamiento

Se creó el archivo de prueba `phpinfo.php`:

```bash
echo '<?php phpinfo(); ?>' > /srv/nginx/html/phpinfo.php
```

<img width="1328" height="843" alt="image" src="https://github.com/user-attachments/assets/b05801a5-58d5-4182-948f-34a47dabe62a" />

Se configuró el reenvío de puertos en VirtualBox (host 8080 → guest 80) y se accedió desde el navegador a:

```
http://127.0.0.1:8080/phpinfo.php
```

El resultado mostró correctamente la página de información de PHP 8.4.7 con Server API: FPM/FastCGI, confirmando la comunicación exitosa entre NGINX y PHP-FPM.


Conclusiones

Durante el desarrollo del proyecto se logró implementar un entorno web completamente funcional utilizando NGINX y PHP-FPM compilados desde código fuente. 

Esta práctica permitió comprender el proceso manual de compilación, instalación y configuración de servicios en Linux, además del manejo de SystemD y comunicación FastCGI mediante sockets UNIX.

También se reforzaron conocimientos relacionados con administración de servidores, permisos de usuarios y optimización de servicios web. 

Finalmente, se verificó el correcto funcionamiento del servidor mediante la ejecución de scripts PHP desde un navegador web.


## Bibliografía

nginx. (2026). *nginx 1.31.1 release*. nginx.org. https://nginx.org/en/download.html

PHP Group. (2026). *PHP 8.4.7 release*. php.net. https://www.php.net/downloads.php

AlmaLinux OS Foundation. (2026). *AlmaLinux OS 10.1 documentation*. almalinux.org. https://almalinux.org/

Red Hat, Inc. (2026). *Using systemd unit files*. Red Hat Documentation. https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10

Oracle Corporation. (2026). *Oracle VM VirtualBox user manual*. virtualbox.org. https://www.virtualbox.org/manual/

nginx. (2026). *Beginner's guide — FastCGI*. nginx.org. https://nginx.org/en/docs/beginners_guide.html

PHP Group. (2026). *PHP-FPM configuration*. php.net. https://www.php.net/manual/en/install.fpm.configuration.php


