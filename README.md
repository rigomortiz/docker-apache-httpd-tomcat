# Docker con Apache httpd y Tomcat

**Apache como Frontend de Tomcat**

Usar [Apache server como frontend](https://docs.jelastic.com/tomcat-behind-apache/) agrega mejoras como son:

 * Alta disponibilidad mediante la realización de equilibrios de carga entre varios servidores tomcat.
 * Procesamiento rápido y entrega de contenido estático. 
 * Mejoras de seguridad. 
 * Funciones adicionales a través de módulos Apache.  

En este ejemplo vamos a tener un contenedor docker para conectar 2 servidores Tomcat con dos war corriendo cada uno a un Apache httpd.  


## Proyecto

Primero tenemos que tener instalado Docker, puedes encontrar las instrucciones de instalación desde el [sitio oficial](https://docs.docker.com/engine/install/). También necesitamos instalar [Docker Compose ](https://docs.docker.com/compose/).

#### Repositorio
https://github.com/rigobertocanseco/docker-apache-httpd-tomcat


### Directorio

El proyecto se encuentra organizado de la siguiente manera

```
├── docker-compose.yml
├── .env
├── .dockerignore
├── .gitignore
├── httpd
│    ├── certs
│    │   ├── localhost.crt
│    │   ├── localhost.key
│    │   ├── rootCA.crt
│    │   └── rootCA.key
│    ├── 000-default.conf
│    ├── apache2.conf
│    ├── Dockerfile
│    ├── jk.conf
│    └── worker.properties
├── tomcat1
│    ├── Dockerfile
│    ├── sample.war
│    └── server.xml
└── tomcat2
     ├── Dockerfile
     ├── sample.war
     └── server.xml
```

1. **docker-compose.yml**: Aquí tenemos la configuración para nuestro contenedor Docker 
2. **env**: Nuestras variables de entorno para nuestro contenedor
3. **.dockerignore**: Archivos a ignorar 
4. **.ignore**: Archivos a ignorar 
5. **httpd**: Directorio con la configuración para el contenedor con Apache httpd
6. **certs**: Directorio con los certificados SSL
    * Certificado del servidor(localhost.crt) y llave privada(localhost.key).
    * Certificado CA y llave privada (rootCA.crt y rootCA.key).
7. **tomcat1**: Directorio con la configuración para el contenedor con Apache Tomcat 
8. **tomcat2**: Directorio con la configuración para el contenedor con Apache Tomcat  

### Configuración del servidor Apache

#### Dockerfile
Vamos a revisar las partes principales del archivo:

* Creamos un contenedor con una imagen de ubuntu
```dockerfile
FROM ubuntu:latest
```

* Definimos nuestras variables de entorno   
```dockerfile
ENV DEBIAN_FRONTEND="noninteractive"
ENV SITE_NAME rigobertocanseco.dev
ENV URL_SECURE https://${SITE_NAME}:8443/

# Certificates of demo
ENV APACHE_SSL_CERT_CA rootCA.crt
ENV APACHE_SSL_CERT localhost.crt
ENV APACHE_SSL_PRIVATE localhost.key

# Certificates of prod
#ENV APACHE_SSL_CERT_CA ca_bundle.crt
#ENV APACHE_SSL_CERT certificate.crt
#ENV APACHE_SSL_PRIVATE private.key
```

* Instalamos Apache con el comando `apt` es responsable de la administración de paquetes y, por lo tanto, de la instalación.
```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends apache2 libapache2-mod-jk
``` 

* Agregamos los archivos para configurar Apache 
```dockerfile
ADD apache2.conf /etc/apache2/apache2.conf
ADD 000-default.conf /etc/apache2/sites-enabled/000-default.conf
ADD worker.properties /etc/libapache2-mod-jk/workers.properties
ADD jk.conf /etc/apache2/mods-available/jk.conf
``` 

* Copiamos nuestros certificados SSL al contenedor
```dockerfile
COPY certs/${APACHE_SSL_CERT_CA} /etc/ssl/certs/${APACHE_SSL_CERT_CA}
COPY certs/${APACHE_SSL_CERT} /etc/ssl/certs/${APACHE_SSL_CERT}
COPY certs/${APACHE_SSL_PRIVATE} /etc/ssl/private/${APACHE_SSL_PRIVATE}
```

* Habilitamos los módulos de Apache que vamos a usar, para obtener más información consulte la documentación del módulo Apache [mod_http2](https://httpd.apache.org/docs/2.4/mod/mod_http2.html).  
```dockerfile
# Modules to apache
RUN a2enmod proxy \
&& a2enmod ssl \
&& a2enmod proxy_ajp \
&& a2enmod rewrite

# Mod Proxy(mod_proxy)
#&& a2enmod proxy_http \
#&& a2enmod proxy_fcgi \
#&& a2enmod proxy_balancer \
#&& a2enmod lbmethod_byrequests \
#&& a2enmod headers
```

* Por último creamos un volumen con el directorio de logs de apache, exponemos el puerto 80 y 443 para el puerto normal y puerto seguro, respectivamente. Y el último comando a ejecutar es para poner en escucha el nuevo servicio HTTP de Apache. 

```dockerfile
VOLUME ["/var/log/apache2"]
EXPOSE 80 443
CMD ["apachectl", "-k", "start", "-DFOREGROUND"]
```

#### 000-default.conf

Vamos a ver las partes principales del archivo, [aquí](https://httpd.apache.org/docs/2.4/en/mod/core.html) se encuentra la documentación de las directivas de configuración de apache.


* Configuramos el servidor web apache httpd, se redireccionan las peticiones al puerto seguro 443
```apache
<VirtualHost *:80>
...
        # Redirect to port secure
        #Redirect / ${URL_SECURE}
        RewriteEngine on
        ReWriteCond %{SERVER_PORT} !^443$
        RewriteRule ^/(.*) https://%{HTTP_HOST}/$1 [NC,R,L]
...
</VirtualHost>
```

* Se configura el puerto 443 como puerto seguro, se indican el Certificado SSL, la llave privada y el certificado de CA ([Introducción a SSL](https://httpd.apache.org/docs/2.4/ssl/ssl_intro.html)) así como directivas de configuración disponibles en el módulo **[mod_ssl](https://httpd.apache.org/docs/2.4/mod/mod_ssl.html)**  

```apache
<IfModule mod_ssl.c>
    <VirtualHost *:443>
...
        SSLCertificateFile      /etc/ssl/certs/${APACHE_SSL_CERT}
        SSLCertificateKeyFile /etc/ssl/private/${APACHE_SSL_PRIVATE}
        SSLCACertificatePath /etc/ssl/certs/

        SSLEngine on
        SSLProxyEngine On
        SSLProxyVerify none
        SSLProxyCheckPeerCN Off
        SSLProxyCheckPeerName Off
        SSLProxyCheckPeerExpire Off
        ProxyPreserveHost Off
        ProxyRequests Off
...		
    </VirtualHost>
</IfModule>
```

* Se indica el contexto de un worker de Tomcat, donde **sample** es un contexto para el worker **loadbalancer**, y **status** para el worker **status**. La documentación de JK Connector se encuentra [aquí](https://tomcat.apache.org/tomcat-4.1-doc/config/jk.html), y la documentación de los workes la puedes encontrar [aquí](https://tomcat.apache.org/connectors-doc/common_howto/workers.html). 

```apache
<IfModule mod_ssl.c>
    <VirtualHost *:443>
        ...
        
                jkMount /sample  loadbalancer
                JkMount /sample/* loadbalancer
        
                JkMount /status   status
                JkMount /status/*   status
        
                #<Location /server-status>
                #    SetHandler server-status
                #    Order Deny,Allow
                #    Allow from all
                #</Location>
        
                #Location /balancer-manager>
                #    SetHandler balancer-manager
                #    Order Deny,Allow
                #    Allow from all
                #</Location>
                #Header add Set-Cookie "ROUTEID=.%{BALANCER_WORKER_ROUTE}e; path=/" env=BALANCER_ROUTE_CHANGED
        
                #<Proxy balancer://tomcat-cluster>
                #    BalancerMember ajp://tomcat1:9090 route=tomcat1_1  retry=0
                #    BalancerMember ajp://tomcat2:9091 route=tomcat1_2  retry=0
                #    BalancerMember ajp://tomcat1:9090 route=tomcat2_1  retry=0
                #    BalancerMember ajp://tomcat2:9091 route=tomcat2_2  retry=0
                #</Proxy>
        
                #ProxyPass           /sample   balancer://tomcat-cluster/sample  stickysession=JSESSIONID│jsessionid scolonpathdelim=On
                #ProxyPassReverse    /sample   balancer://tomcat-cluster/sample
    </VirtualHost>
</IfModule>
```

#### apache2.conf

El archivo **apache2.conf** contiene la configuración para el Apache httpd por default([Ver documentación](https://httpd.apache.org/docs/2.4/mod/quickreference.html)).
 

#### jk.config

El archivo **jk.config** contiene la configuración para el módulo [JK Connector](https://tomcat.apache.org/tomcat-4.1-doc/config/jk.html).
El conector JK es un componente de servidor web que se utiliza para integrar de forma "invisible" Tomcat con un servidor web como Apache. La comunicación entre el servidor web y Tomcat se realiza a través del protocolo JK (también conocido como [protocolo AJP](https://en.wikipedia.org/wiki/Apache_JServ_Protocol)).  
```properties
JkLogLevel info
JkOptions +ForwardKeySize +ForwardURICompat -ForwardDirectories
JkRequestLogFormat "%w %V %T"
```
 

#### worker.properties

El archivo **worker.properties** contiene la configuración  para [Tomcat worker](https://tomcat.apache.org/connectors-doc/reference/workers.html). Un worker es una instancia de Tomcat que está esperando para ejecutar servlets o cualquier otro contenido en nombre de algún servidor web. En este archivo es donde vamos a configurar la conexión de nuestras instancias Tomcat con Apache httpd por medio de AJP[(Ver documentación)](https://tomcat.apache.org/connectors-doc/index.html).
 
 * Definimos dos workers loadbalancer y status, el primero va a ser nuestro _balanceador de cargar_ y el segundo lo usaremos para ver el status de nuestro loadbalancer. El loadbalancer se encuentra de la siguiente manera:
    * Tomcat1: 2 workers, tomcat1_1 por puerto 9090 y tomcat1_2 por puerto 9091,  
    * Tomcat2: 2 workers, tomcat2_1 por puerto 9090 y tomcat2_2 por puerto 9091

Los 4 workers están configurados con sticky_session, usan los puertos 9090 y 9091 que configuramos para **AJP13**. Ver la documentación del connector [mod_jk](https://tomcat.apache.org/connectors-doc/reference/apache.html).
```properties
#
#------ worker list ------------------------------------------
#---------------------------------------------------------------------
#
#
# The workers that your plugins should create and work with
#
ps=/
worker.list=loadbalancer,status
#
#
#
#------ DEFAULT LOAD BALANCER WORKER DEFINITION ----------------------
#---------------------------------------------------------------------
#

#
# The loadbalancer (type lb) workers perform wighted round-robin
# load balancing with sticky sessions.
# Note:
#  ----&amp;gt; If a worker dies, the load balancer will check its state
#        once in a while. Until then all work is redirected to peer
#        workers.
worker.loadbalancer.type=lb
worker.loadbalancer.balance_workers=tomcat1_1,tomcat1_2,tomcat2_1,tomcat2_2
worker.loadbalancer.sticky_session=True
worker.status.type=status

#
#------ ajp13_worker WORKER DEFINITION ------------------------------
#---------------------------------------------------------------------
#

#
# Defining a worker named ajp13_worker and of type ajp13
# Note that the name and the type do not have to match.
#
worker.tomcat1_1.port=9090
worker.tomcat1_1.host=host.tomcat1
worker.tomcat1_1.type=ajp13
worker.tomcat1_1.connection_pool_timeout=600
worker.tomcat1_1.route=tomcat1_1
#worker.tomcat1_1.socket_keepalive=1

worker.tomcat1_2.port=9091
worker.tomcat1_2.host=host.tomcat1
worker.tomcat1_2.type=ajp13
worker.tomcat1_2.connection_pool_timeout=600
worker.tomcat1_2.route=tomcat1_2
#worker.tomcat1_2.socket_keepalive=1


worker.tomcat2_1.port=9090
worker.tomcat2_1.host=host.tomcat2
worker.tomcat2_1.type=ajp13
worker.tomcat2_1.connection_pool_timeout=600
worker.tomcat2_1.route=tomcat2_1
#worker.tomcat2_1.socket_keepalive=1

worker.tomcat2_2.port=9091
worker.tomcat2_2.host=host.tomcat2
worker.tomcat2_2.type=ajp13
worker.tomcat2_2.connection_pool_timeout=600
worker.tomcat2_2.route=tomcat2_2
#worker.tomcat2_2.socket_keepalive=1
```
 
 
### Configuración del contenedor con Tomcat1 y Tomcat2

#### Dockerfile
 Vamos a revisar las partes del archivo Dockerfile:

 * Se crea una imagen de tomcat 9.0, 
 * Copiamos el archivo server.xml y el sample.war en app1 y app2. 
 * Por último exponemos los puertos 9090 y 9091. 
 
```dockerfile
FROM tomcat:9.0

COPY server.xml /usr/local/tomcat/conf/

# add context to /usr/local/tomcat/webapps
COPY sample.war /usr/local/tomcat/app1/sample.war
COPY sample.war /usr/local/tomcat/app2/sample.war

EXPOSE 9090 9091
```

#### sample.war
Aquí colocamos un war que va a servir como ejemplo para deployar.

#### server.xml
En el archivo server.xml se encuentra la configuración para el servidor Tomcat, en este caso solo nos interesa revisar lo siguiente:
 * En `<Service name="Catalina1">` se encuentra la un tomcat que corre por el puerto 8080, dentro encontramos `<Connector protocol="AJP/1.3" ... port="9090" .../>`, `<Engine name="Catalina" defaultHost="host.tomcat1" jvmRouter="tomcat1_1">` y `<Host name="host.tomcat1"  appBase="app1" unpackWARs="true" autoDeploy="true">`
 * En `<Service name="Catalina2">` se encuentra la un tomcat que corre por el puerto 8081, dentro encontramos `<Connector protocol="AJP/1.3" ... port="9091" .../>`, `<Engine name="Catalina" defaultHost="host.tomcat1" jvmRouter="tomcat1_2">` y `<Host name="host.tomcat1"  appBase="app2" unpackWARs="true" autoDeploy="true">`
 
En el servidor Tomcat tenemos 2 servicios **Catalina1** y **Catalina2** que corren por el puerto 8080(puerto para AJP 9090) y 8081(puerto para AJP 9091), respectivamente. Cada servicio se identifica con un nombre para **jvmRouter**, tomcat1_1 y tomcat1_2, estos nombres están ligados con `worker.tomcat1_1.route=tomcat1_1` y `worker.tomcat1_2.route=tomcat1_2` de **worker.properties**.

El Hostname para este Tomcat es **host.tomcat1** que también se encuentran en `worker.tomcat1_1.host=host.tomcat1` y `worker.tomcat1_2.host=host.tomcat1`     

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Server port="8005" shutdown="SHUTDOWN">
...
    <Service name="Catalina1">
...

    <Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000"/>
...
    <!-- Define an AJP 1.3 Connector on port 8009 -->
    <Connector protocol="AJP/1.3"
               URIEncoding="UTF-8" enableLookups="false"
               address="0.0.0.0" port="9090" connectionTimeout="20000"
               maxConnections="256" keepAliveTimeout="30000"
               redirectPort="8443" secretRequired="false" />
    
    <Engine name="Catalina" defaultHost="host.tomcat1" jvmRouter="tomcat1_1">
...
      <Host name="host.tomcat1"  appBase="app1" unpackWARs="true" autoDeploy="true">
...
      </Host>
    </Engine>
  </Service>

  <Service name="Catalina2">
...
        <Connector port="8081" protocol="HTTP/1.1" connectionTimeout="20000"/>
...
        <!-- Define an AJP 1.3 Connector on port 8009 -->
        <Connector protocol="AJP/1.3"
                   URIEncoding="UTF-8" enableLookups="false"
                   address="0.0.0.0" port="9091" connectionTimeout="20000"
                   maxConnections="256" keepAliveTimeout="30000"
                   redirectPort="8443" secretRequired="false" />
...
        <Engine name="Catalina" defaultHost="host.tomcat1" jvmRouter="tomcat1_2">
...
            <Host name="host.tomcat1" appBase="app2" unpackWARs="true" autoDeploy="true">

...
            </Host>
        </Engine>
    </Service>

</Server>
```

### Configuración de docker compose

#### .env
Aquí declaramos 2 variables de entorno para los puertos que vamos a utilizar  
```properties
HOST_HTTP_PORT=8080
HOST_HTTPS_PORT=8443
```

#### docker-compose.yml
Tenemos configurado  2 servidores Tomcat y el servidor Apache httpd en un Docker Compose ([Documentación](https://docs.docker.com/compose/compose-file/compose-file-v2/)). 
* Asignamos un volumen para el directorio **log**
* Tenemos asignado el puerto 8080 para el puerto 80 de Apache y el puerto 8443 para el puerto seguro 443. 
* Ligamos los dos servicios Tomcat **tomcat1** y **tomcat2** al contenedor de Apache httpd con los hostnames **host.tomcat1** y **host.tomcat2**.  

```yaml
version: '2'
services:
  tomcat1:
    build: tomcat1/.
  tomcat2:
    build: tomcat2/.
  httpd:
    volumes:
      - ./logs:/var/log/apache2
    ports:
      - "${HOST_HTTP_PORT}:80"
      - "${HOST_HTTPS_PORT}:443"
    build: httpd/.
    links:
      - tomcat1:host.tomcat1
      - tomcat2:host.tomcat2
```

### Ejecutar servicios

Los comandos de compilación y ejecución de Docker deben ejecutarse desde la raíz del directorio del proyecto después de clonar este repositorio.

````shell script
$ docker-compose build
$ docker-compose up -d
$ docker-compose ps
````

Usando el comando **docker-compose ps** deberíamos poder ver el nuevo contenedor en la lista, como se indica a continuación.
````
                  Name                                Command               State                      Ports                   
-------------------------------------------------------------------------------------------------------------------------------
docker-apache-httpd-tomcat-ssl_httpd_1     apachectl -k start -DFOREG ...   Up      0.0.0.0:8443->443/tcp, 0.0.0.0:8080->80/tcp
docker-apache-httpd-tomcat-ssl_tomcat1_1   catalina.sh run                  Up      8080/tcp, 9090/tcp, 9091/tcp               
docker-apache-httpd-tomcat-ssl_tomcat2_1   catalina.sh run                  Up      8080/tcp, 9090/tcp, 9091/tcp   
````

Editamos el hostname de nuestra máquina. En linux editamos el archivo **/etc/hosts** agregamos lo siguiente. `127.0.0.1       demo.com`

A partir de este momento es posible acceder a https://demo.com:8443/sample/
 y para ver el balanceador de carga https://demo.com:8443/status/


### Links
[Tomcat Clustering - A Step By Step Guide](https://www.mulesoft.com/tcat/tomcat-clustering)  
[Comparing mod_proxy and mod_jk](https://developer.jboss.org/people/mladen.turk/blog/2007/07/16/comparing-modproxy-and-modjk)  
[Tomcat clustering](https://www.ramkitech.com/search/label/tomcat%20clustering)  
 [mod_jk, mod_proxy and mod_proxy_ajp](https://javafatihk.blogspot.com/2014/11/modjk-modproxy-and-modproxyajp.html)  
 [APACHE HTTP SERVER CONNECTORS AND LOAD BALANCING GUIDE](https://access.redhat.com/documentation/en-us/red_hat_jboss_core_services/2.4.23/html-single/apache_http_server_connectors_and_load_balancing_guide/index)
 