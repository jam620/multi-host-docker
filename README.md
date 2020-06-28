# Docker Multi-Site con Let's Encrypt

El uso de contenedores ha facilitado la implementación de diversas herramientas con variados stacks de aplicaciones en un solo host, pero la pregunta sobre las peticiones y como distribuirlas siempre ha sido por medio de proxy como en el articulo del colega de la Comunidad Dojo [Elzer Pineda](https://backtrackacademy.com/articulo/virtual-host-docker-redirect-http-to-https-apache "Articulo"). De igual manera con la configuración de un nginx podemos obtener los mismos resultados, sin embargo me preguntaba ¿Porqué no hacerlo con los contenedores? 

En el siguiente articulo les explicaré como realizar una configuración básica con 2 contenedores de diversas herramientas web y sus respectivos certificados con lets encrypt.

##### Requerimientos

* VPS en [Vultr.com](https://www.vultr.com/?ref=8403796-6G "Referencia 100")  con 2GB de Ram, 1 vcpu, 55 GB SSD

* Dominio gratuito o pagado en  este caso usare 2  subdominios *.utponline.xyz
* Contar con docker y docker compose instalado previamente



1. #### Arquitectura Deseada

   El siguiente diagrama nos permite utilizar un Nginx  [Proxy](https://github.com/nginx-proxy/nginx-proxy "Github"),  que repartirá las peticiones de los subdominios a los respectivos contenedores con wordpress y Ghost. El contenedor de lets encrypt lo veremos en el archivo docker-compose.yml

   <img src="https://github.com/jam620/multi-host-docker/tree/master/img/1.png" alt="1" style="zoom:50%;" />

   Fuente: Propia, el contenedor de lets encrypt no esta reflejado en el gráfico dado su utilidad es para generar el certificado únicamente.

2. #### Ahora vamos a compartir el archivo docker-compose.yml  

   ```yml
   version: '3.1'
   services:
       nginx-proxy:
           image: jwilder/nginx-proxy
           ports:
            - "80:80"
            - "443:443"
           volumes:
             - /var/run/docker.sock:/tmp/docker.sock:ro
             - ./certs:/etc/nginx/certs:ro
             - ./vhostd:/etc/nginx/vhost.d
             - ./html:/usr/share/nginx/html
           labels:
             - com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy
   
       letsencrypt:
           image: jrcs/letsencrypt-nginx-proxy-companion:v1.13
           restart: always
           environment:
           - NGINX_PROXY_CONTAINER=nginx-proxy
           volumes:
             - ./certs:/etc/nginx/certs:rw
             - ./vhostd:/etc/nginx/vhost.d
             - ./html:/usr/share/nginx/html
             - /var/run/docker.sock:/var/run/docker.sock:ro
   
       wordpress:
           image: wordpress
           container_name: wordpress_1
           links:
            - mariadb:mysql
           expose:
            - 80
           environment:
            - WORDPRESS_DB_PASSWORD=qwertyqwerty
            - WORDPRESS_DB_USER=userwp
            - WORDPRESS_DB_NAME=wpdb
            - "VIRTUAL_HOST=web1.utponline.xyz"
            - "LETSENCRYPT_HOST=web1.utponline.xyz"
            - "LETSENCRYPT_EMAIL=jose.moreno@comunidaddojo.org"
           volumes:
            - ./web1/html:/var/www/html
   
       mariadb:
           image: mariadb
           environment:
            - MYSQL_ROOT_PASSWORD=qwerty123456789
            - MYSQL_DATABASE=wpdb
            - MYSQL_USER=userwp
            - MYSQL_PASSWORD=qwertyqwerty
           volumes:
            - ./database:/var/lib/mysql
   
       ghost:
           image: ghost 
           expose:
            - 80
           environment:
               - VIRTUAL_HOST=ghost.utponline.xyz
               - VIRTUAL_PORT=2368
               - "LETSENCRYPT_HOST=ghost.utponline.xyz"
               - "LETSENCRYPT_EMAIL=jose.moreno@comunidaddojo.org"
   
   ```

   * Los parámetros que debemos tomar en cuanta para la creación delos certificados son los siguientes:
     - VIRTUAL_HOST: dominio de la aplicación a utilizar 
     - LETSENCRYPT_HOST: el parámetro toma el host que vamos a generar los certificados
     - LETSENCRYPT_EMAIL: correo a utilizar para el certificado
     - NGINX_PROXY_CONTAINER: definimos el proxy 
     - labels: la etiqueta que vamos a utilizar para indicar el proxy utilizado con el contenedor de lets encrypt

3. #### Inicialización de Arquitectura

   Para iniciar nuestro docker multi-site configurado con nuestros contenedores y certificado utilizamos el siguiente comando: 

   ```yml
   docker-compose up -d
   ```

   ![2](https://github.com/jam620/multi-host-docker/tree/master/img/2.png)

   Verificamos que los contenedores hayan iniciado correctamente

   ![3](https://github.com/jam620/multi-host-docker/tree/master/img/3.png)

   

   Observamos de igual manera los directorios creados por el docker-compose.yml 

   <img src="https://github.com/jam620/multi-host-docker/tree/master/img/4.png" alt="4" style="zoom:50%;" />

   

4. #### Resultado final

* Contenedor Wordpress con el dominio https://web.utponline.xyz

  <img src="https://github.com/jam620/multi-host-docker/tree/master/img/5.png" alt="5" style="zoom:50%;" />

  

* Contenedor con blog ghost y el dominio http://ghost.utponline.xyz

  <img src="https://github.com/jam620/multi-host-docker/tree/master/img/6.png" alt="6" style="zoom:50%;" />



​	Certificado generado con lets encrypt 

​	<img src="https://github.com/jam620/multi-host-docker/tree/master/img/7.png" alt="7" style="zoom: 33%;" />

Hemos automatizado la creación de certificados, además de utilizar un proxy que administra las peticiones a los contenedores con diversas tecnologías o herramientas web, adicionalmente se podría añadir  a esta arquitectura un contenedor con gitlab, jenkins, entre otros.

#### Referencias 

* https://github.com/nginx-proxy/nginx-proxy
* https://github.com/pablokbs/peladonerd/blob/master/varios/1/docker-compose.yaml
* https://blog.florianlopes.io/host-multiple-websites-on-single-host-docker/