# Tecnológico de Estudios Superiores del Oriente del Estado de México

## Reporte : Proyecto NGINX y PHP

* **Carrera:** Ingeniería en Sistemas Computacionales
* **Materia:** Administración de Servidores
* **Integrantes:**
1.  Palomares Morales Fernando Eduardo
2.  Ramos Rojas Ofelia Atziri

* **Profesor:** Gustavo Moises Romero Gonzalez
* **Fecha:** 25 Mayo 2026

---

## 1. Objetivo General
Construir una arquitectura web de alto rendimiento mediante la autogestión de binarios desde el código fuente original en AlmaLinux 9. El propósito es establecer un stack NGINX/PHP 8.4 totalmente optimizado, centralizado en /srv/nginx y comunicado vía Sockets UNIX, asegurando la resiliencia mediante el control absoluto de los servicios del sistema.

## 2. Objetivos Específicos
Gestión de Origen: Compilar el servidor NGINX y el motor PHP 8.4 desde sus archivos fuente, asegurando una instalación limpia bajo un directorio único de trabajo.

Integración Especializada: Habilitar librerías críticas de procesamiento multimedia y localización (GD, WebP, Intl) durante la etapa de compilación.

Optimización de I/O: Desplazar la comunicación entre el servidor web y el procesador de scripts hacia Sockets UNIX para evitar latencias de red en el stack TCP/IP.

Persistencia de Servicio: Integrar ambos componentes en SystemD, garantizando que el entorno esté disponible automáticamente tras cada arranque del sistema operativo.

Aseguramiento de Operatividad: Gestionar las políticas de seguridad de SELinux para permitir la ejecución de procesos compilados externamente, garantizando la estabilidad del entorno.

---

## 3. Desarrollo del Proyecto

### A. Preparación de Dependencias del Sistema
Se preparó el sistema base instalando herramientas de compilación y librerías necesarias::
```bash
sudo dnf update -y
sudo dnf groupinstall "Development Tools" -y
sudo dnf install -y epel-release wget pcre-devel pcre2-devel zlib-devel openssl-devel \
    libxml2-devel sqlite-devel libpng-devel libjpeg-turbo-devel freetype-devel \
    libwebp-devel libcurl-devel libffi-devel libicu-devel systemd-devel oniguruma-devel
```
Se crearon los usuarios de sistema::
```bash
sudo groupadd nginx
sudo useradd -g nginx -s /sbin/nologin -r nginx
sudo useradd -g nginx -s /sbin/nologin -r php
```
Se descargó el código fuente estable de NGINX, se asignó el Prefix de instalación en la ruta solicitada y se ejecutó la compilación:
```bash
cd /usr/src
sudo wget [http://nginx.org/download/nginx-1.26.0.tar.gz](http://nginx.org/download/nginx-1.26.0.tar.gz)
sudo tar -xzvf nginx-1.26.0.tar.gz
cd nginx-1.26.0
sudo ./configure \
    --prefix=/srv/nginx \
    --user=nginx \
    --group=nginx \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-threads \
    --with-file-aio
sudo make && sudo make install
```
Se compiló NGINX 1.26.0 configurado en /srv/nginx con módulos de SSL y Threading. Posteriormente, se compiló PHP 8.4.1 con las extensiones requeridas:
```bash
cd /usr/src
sudo wget [https://www.php.net/distributions/php-8.4.1.tar.gz](https://www.php.net/distributions/php-8.4.1.tar.gz)
sudo tar -xzvf php-8.4.1.tar.gz
cd php-8.4.1
sudo ./configure \
    --prefix=/srv/nginx \
    --mandir=/srv/nginx/man \
    --enable-fpm \
    --with-fpm-user=php \
    --with-fpm-group=nginx \
    --with-mysqli \
    --with-pdo-mysql \
    --with-openssl \
    --with-zlib \
    --with-curl \
    --enable-gd \
    --with-jpeg \
    --with-webp \
    --with-freetype \
    --enable-intl \
    --enable-mbstring \
    --enable-soap \
    --enable-calendar
sudo make -j4 && sudo make install
```
Se configuró /srv/nginx/etc/php-fpm.d/www.conf para escuchar en el socket:
```bash
[www]
user = php
group = nginx
listen = /tmp/php84.sock
listen.owner = php
listen.group = nginx
listen.mode = 0660
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```
En el archivo /srv/nginx/conf/nginx.conf se direccionó la interpretación de scripts PHP hacia el descriptor del socket UNIX:
```bash
location ~ \.php$ {
    fastcgi_pass   unix:/tmp/php84.sock;
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include        fastcgi_params;
}
```
Se crearon los scripts de servicio tipo Slice en /etc/systemd/system/ para nginx.service y php-fpm8.4.service.

Al reiniciar la máquina virtual se detectó que el servicio fallaba con el código de error status=203/EXEC. 
Se diagnosticó que SELinux bloqueaba el entorno al estar en modo estricto. Para solucionarlo de forma definitiva se configuró el estado de SELinux en modo permisivo editando el archivo /etc/selinux/config:
```bash
SELINUX=permissive
```
Posteriormente, se recargaron los daemons y se habilitó el auto-arranque en el sistema:
```bash
sudo systemctl daemon-reload
sudo systemctl enable nginx.service php-fpm8.4.service
sudo systemctl start php-fpm8.4.service nginx.service
```

Aquí tienes una versión más sencilla y clara de tu conclusión, manteniendo los puntos clave pero con un lenguaje más directo:

### 4. Conclusiones

Compilar nuestros propios servidores (NGINX y PHP) desde el código fuente nos permitió tener un control total sobre las herramientas instaladas, logrando que funcionen de manera más eficiente al incluir solo lo necesario. Al configurar la comunicación entre ambos mediante Sockets UNIX en lugar de puertos de red, logramos que los datos viajen más rápido dentro del sistema, mejorando el rendimiento general. Finalmente, este proyecto nos ayudó a entender mejor cómo funcionan los permisos de seguridad en sistemas como AlmaLinux (SELinux), aprendiendo que es fundamental configurar correctamente los accesos para que las aplicaciones puedan ejecutarse sin bloqueos.

# 5. Bibliografía 
AlmaLinux OS Foundation. (2025). AlmaLinux OS 9 Deployment and Configuration Guide. AlmaLinux Docs. https://docs.almalinux.org/

The Nginx Organization. (2026). Nginx Documentation: Core installation from sources. Nginx Enterprise. https://nginx.org/en/docs/install.html

The PHP Group. (2026). PHP Manual: FastCGI Process Manager (FPM) Configuration. PHP Documentation. https://www.php.net/manual/en/install.fpm.configuration.php
