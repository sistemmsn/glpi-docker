## Implementacion de glpi en docker 


Creación del Dockerfile de glpi:

![Docker1](/images/docker.png)

- Nota: Conuslta tu zona horaria en: [Timenzones](https://www.w3schools.com/php/php_ref_timezones.asp).

### Dockerfile de glpi

```markdown
# Correremos este comando para construir nustra imagen personalizada

docker build . -t apache-glpi:sistemmsn -f apache-glpi --progress=plain

# El parametro -f es para indicarle el nombre de nuestro Dockerfile y el -t es nuestro tag de versiones 

```


```markdown
FROM almalinux:latest

LABEL vendor=sistemmsn

ENV DUMBINIT 1.2.5
ENV GLPIV 10.0.5

RUN dnf update -y

RUN dnf install -y wget mysql git vim sudo httpd python39

RUN dnf install -y http://rpms.remirepo.net/enterprise/remi-release-8.rpm && \
    dnf module -y install php:remi-8.1

RUN dnf -y install php \
    php-mysql \
    php-pdo \
    php-gd \
    php-mbstring  \
    php-imap \
    php-ldap \
    php-pecl-zendopcache \
    php-pecl-apcu \
    php-xmlrpc \
    php-imap \
    php-opcache \
    php-pecl-apcu  \
    php-intl \
    php-zip \
    php-sodium\
    php-imagick \
    php-fpm \
    htop

RUN dnf clean all


#Aplicando congfuraciones para el funcionamiento del servicio apache y php-fpm
RUN sed -i 's|max_execution_time = 30|max_execution_time = 600|'  /etc/php.ini && \
    sed -i 's|Listen 80|Listen 88|'  /etc/httpd/conf/httpd.conf  && \
    sed -i 's|DocumentRoot "/var/www/html"|DocumentRoot "/var/www/glpi"|'  /etc/httpd/conf/httpd.conf  && \
    sed -i 's|session.cookie_httponly =|session.cookie_httponly = 1|' /etc/php.ini && \
# Favor de reemplazar su zona horaria
    sed -i 's|;date.timezone =|date.timezone = America/Mexico_City|' /etc/php.ini && \
    sed -i 's/^#ServerName www.example.com:80/ServerName www.example.com/g' /etc/httpd/conf/httpd.conf


#Creando directorios
RUN mkdir -p /run/php-fpm && \
    mkdir -p /etc/glpi && \
    mkdir -p /var/lib/glpi && \
    mkdir -p /var/log/glpi

RUN cd /var/www && \
    wget https://github.com/glpi-project/glpi/releases/download/${GLPIV}/glpi-${GLPIV}.tgz && \
    tar -xvzf glpi-${GLPIV}.tgz && \
    rm -rf glpi-${GLPIV}.tgz && \
    chown apache:apache -R /var/www/glpi && \
    cd

RUN python3.9 -m venv yacronenv && \
    . yacronenv/bin/activate && \
    pip install yacron && \
    ln -s /yacronenv/bin/yacron /usr/sbin/yacron

RUN wget -O /usr/local/bin/dumb-init https://github.com/Yelp/dumb-init/releases/download/v${DUMBINIT}/dumb-init_${DUMBINIT}_x86_64 && \
    chmod +x /usr/local/bin/dumb-init

RUN echo "#!/bin/bash" >> /etc/rc.local && \
    echo "php-fpm -F &" >> /etc/rc.local && \
    echo "httpd  -k start &" >> /etc/rc.local && \
    echo "yacron -c /yacronenv/crontab.yaml" >> /etc/rc.local &&  \
    chmod 755 /etc/rc.local

RUN printf '%s\n' > /yacronenv/crontab.yaml \
    'jobs:' \
    '  - name: glpi'  \
    '    command: php /var/www/glpi/front/cron.php' \
    '    schedule: "*/2 * * * *"'

RUN mkdir /var/lib/glpi/_cron && \
    mkdir /var/lib/glpi/_dumps && \
    mkdir /var/lib/glpi/_graphs && \
    mkdir /var/lib/glpi/_lock && \
    mkdir /var/lib/glpi/_pictures && \
    mkdir /var/lib/glpi/_plugins && \
    mkdir /var/lib/glpi/_rss && \
    mkdir /var/lib/glpi/_sessions && \
    mkdir /var/lib/glpi/_tmp && \
    mkdir /var/lib/glpi/_uploads && \
    mkdir /var/lib/glpi/_cache

RUN printf "%s\n" > /etc/glpi/local_define.php \
    "<?php" \
    "define('GLPI_VAR_DIR', '/var/lib/glpi');"  \
    "define('GLPI_LOG_DIR', '/var/log/glpi');"

RUN printf "%s\n" > /var/www/glpi/inc/downstream.php \
    "<?php" \
    "define('GLPI_CONFIG_DIR', '/etc/glpi');"  \
    "if (file_exists(GLPI_CONFIG_DIR . '/local_define.php')) {" \
    "   require_once GLPI_CONFIG_DIR . '/local_define.php';" \
    "}"

RUN chown -R apache:apache /etc/glpi && \
    chown -R apache:apache /var/www/glpi/inc && \
    chown -R apache:apache /var/log/glpi && \
    chown -R apache:apache /var/lib/glpi/

EXPOSE 88

VOLUME /var/www/glpi

ENTRYPOINT ["/usr/local/bin/dumb-init", "--"]

CMD  ["/usr/bin/bash", "-c", "/etc/rc.local" ]
```

### Dockerfile de ngix

```markdown

# Correremos este comando para construir nustra imagen personalizada

docker build . -t glpi-nginx:sistemmsn -f glpi-nginx --progress=plain

```

```markdown

FROM almalinux:latest

RUN dnf update -y && \
    dnf install epel-release -y && \
    dnf install -y nginx certbot  python3-certbot-nginx sudo vim && \
    dnf clean all 

RUN ln -sf /dev/stdout /var/log/nginx/access.log && \
	ln -sf /dev/stderr /var/log/nginx/error.log


RUN sudo tee /etc/nginx/conf.d/glpi.conf <<EOF
server {
        server_name Su servidor de domino oh sub;
        listen 80


        location / {
            proxy_pass http://glpi:88/;
            index index.php index.html index.htm;
            proxy_buffering off;
            proxy_http_version 1.1;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $http_connection;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Server $host;
            proxy_set_header X-Forwarded-Proto $scheme;
            access_log off;
            client_max_body_size 1g;
            access_log off;

        }
}
EOF

EXPOSE 443 80

CMD ["nginx", "-g", "daemon off;"]
```


- Nota: el nginx es opcional esto es siempre y cuando ya tenga un proxy levantado, ya que con el nginx estaras certificando tu servidor y #Let's Encrypt requiere del puerto 80 para la certifiación de sus servidores 


![Compose1](/images/compose.jpg)

```markdown
# Correremos este comando para construir nustro stack 

docker compose up -d 

```

### Este es nuestro docker compose para crea nuestro stack de glpi.


- Nota: El stack esta configurado con volumenes no persistences entonces cuando ustedes eliminen el stack para una actualización esta eliminara los datos cosulte: [Persistencia de datos](https://plataforma.josedomingo.org/pledin/cursos/openshift/curso/u05/).

```markdown
version: "3.9"

services:
#MariaDB Container
  mariadb:
    image: mariadb:latest
    container_name: glpi-mariadb
    hostname: glpi-mariadb
    volumes:
      - db:/var/lib/mysql
    environment:
      MARIADB_ROOT_PASSWORD: Glp12023.
      MARIADB_HOSTNAME: glpi-mariadb
      MARIADB_DATABASE: glpidb
      MARIADB_USER: glpiuser
      MARIADB_PASSWORD: glpipass
    restart: always

#GLPI Container
  glpi:
    image: apache-glpi:sistemmsn
    container_name : glpi
    hostname: glpi
    volumes:
      - glpidata:/var/www/html/glpi
    restart: always
#NIGNX Container
  nginx:
    image: nginx-glpi:sistemmsn
    ports:
      - 80:80
      - 443:443
volumes:
    db: {}
    glpidata: {}
```

- Nota: una vez lenvantado el stack de glpi entraremos al contendor de nginx y correremos el siguente comando: certbot --nginx -d glpi.empresa.com una vez tomado el certificado prodeceremos a reinicarle el nginx: nginx -s reload

<h2>Referencias</h2>

[GitHub Yelp](https://github.com/Yelp/dumb-init).

[Ejemplos de ejecución](https://pub.nethence.com/docker/dumb-init).

[GitHub Gjcarneiro](https://github.com/gjcarneiro/yacron).
