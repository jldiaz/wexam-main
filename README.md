# WeXaM

WeXaM es una aplicación para la creación y gestión de preguntas para exámenes, multiusuario y colaborativa.

Consta de varios servicios que pueden desplegarse en diferentes máquinas, pero aquí se suministran instrucciones para hacerlo en una sola máquina mediante `docker-compose`.

# Instalación

## Repositorios Git

Tras descargarse el repositorio `wexam-main` cuyo `README` estás leyendo, debes descargar también los siguientes, organizados en una estructura de carpetas como la que se indica:

```
wexam
├── wexam-main          # Instrucciones generales, docker-compose y configuración ngix
├── wexam-server        # Servidor de la API REST (en python) y base de datos
├── wexam-latex         # Worker para lanzar conversiones de exámenes a LaTeX (opcional)
├── latex-online        # Servicio Web que compila LaTeX, generando pdf (opcional)
└── wexam-angular       # Cliente single-page-application (HTML+JS)
```

Esta estructura se consigue con los siguientes comandos:

```
mkdir wexam
cd wexam
git clone https://github.com/jldiaz/wexam-main
git clone https://github.com/jldiaz/wexam-server
git clone https://github.com/jldiaz/wexam-latex
git clone https://bitbucket.org/JavierGlezRodriguez/wexam wexam-angular
# Opcionalmente (en Amazon AWS EC2 no cabe en una instancia t2.micro)
git clone https://github.com/jldiaz/latex-online
```


## Herramientas

Debes instalar [Docker][1] y [docker-compose][2].

En AWS EC2 el primero se instala con `apt install docker.io` y el segundo con el script que dan en [su página web][2].

Entra en la carpeta `wexam-main`

Examina el fichero `docker-compose.yml`, quizás necesites adaptarlo a tu máquina en la configuración de `nginx`, pues monta el volumen `/etc/letsencrypt` para que tenga acceso a los certificados. En otra máquina estos certificados podrían estar en otro lugar, y sería necesario también modificar `wexam-main/nginx/nginx.conf` para cambiar el nombre DNS de la máquina y la ruta hacia los certificados. 

Esto podría parametrizarse en el futuro con variables de entorno.

El fichero `docker-compose.yml` suministrado es apto para desplegar todos los componentes, salvo latex-online, en una instancia t2.micro de Amazon, y deja un servidor nginx escuchando en el puerto 80 para que redirija el tráfico al 443 por HTTPS, siendo necesario por tanto un certificado para el nombre DNS del nodo (que puede obtenerse con Letsencrypt).

Si se dispone de una instancia más potente (ej: t2.medium) o de un servidor propio, se puede editar `docker-compose.yaml` y seguir los comentarios en él para desplegar también el servicio `latex-online`, que es el más pesado en cuanto a uso de disco porque contiene una instalación completa de TeXLive.

## Despliegue

Edita `docker-compose.yml` y cambia `miclavesecreta` por otra cosa. Será la clave root de la base de datos. No hagas commit de ese cambio pues entonces la clave sería visible en el repositorio.

Edita `../wexam-server/wexam/settings/default.yml` y configura el usuario y contraseña de gmail que usará wexam-server para enviar correos a los usuarios, así como el token secreto para JWT, y si lo estimas necesario el tiempo de caducidad de los token JWT.

Los siguientes comandos crearán las imágenes Docker (la primera vez puede llevar un tiempo ya que tiene que descargarse de internet muchas bibliotecas), y arrancarán todos los servicios en el orden apropiado.

```
docker-compose build
docker-compose up -d
```

Esto lanzará los siguientes servicios (que son también los nombres de los correspondientes contenedores docker):

* `nginx` servidor web escuchando por defecto en el puerto 8088, bajo protocolo HTTPS, que sirve los contenidos (estáticos) de la carpeta `../wexam-angular/dist` y a la vez hace de proxy para el servidor python que implementa la API REST con la que se comunica el cliente angular. Cuando recibe peticiones a la ruta `/wexam-api`, las redirige al servicio `wexam-server`, que es donde está escuchando python (en el puerto 29000).
* `wexam-server` es el servidor python de la API, escuchando en el puerto 29000. No se puede intentar conectar directamente con él, pues está gestionado por el proceso `uwsgi` que no usa protocolo HTTP, sino uno propio. Debe hacerse a través del servidor `nginx`, el cual transforma las peticiones HTTP(s) a la ruta `/wexam-api` al protocolo apropiado antes de pasárselas a uwsgi.
* `wexam-db` es el servicio base de datos. Utiliza postgresql y tiene montado como volumen la carpeta local `/var/postgres` donde está la base de datos para que se persistente (de lo contrario se borraría al parar el contenedor). A esta base de datos se conecta `wexam-server` para gestionar profesores, círculos, preguntas, exámenes, etc. La conexión va autenticada por una clave que está visible en `docker-compose.yml`, por lo que debería cambiarse esa clave (en ese fichero) antes de desplegar.
* `wexam-redis` es un servidor redis usado como cola de trabajos para comunicar `wexam-server` con `wexam-latex`.
* `wexam-latex` tiene un worker vigilando la cola de trabajos de `wexam-redis` y cuando llega uno lo ejecuta. Solo tiene un proceso e hilo por lo que los trabajos se procesan en secuencia. Los trabajos pueden ser convertir JSON en zip o en pdf. Para lo segundo usa el servicio `wexam-pdf`. Almacena una caché de trabajos ya procesados y la usa para no procesarlos de nuevo (se basa en el hash del JSON que recibe y del tipo de conversión a realizar)

En caso de que se haya editado convenientemente `docker-compose.yml` para que lance también `wexam-pdf`, se tendrá un servicio más:

* `wexam-pdf` es una versión propia de latex-online, que escucha peticiones REST a través de las que recibe un archivo latex o un `tgz` que contenga varios archivos latex y los compila devolviendo el pdf. Tiene su propia cache de documentos ya compilados, aunque probablemente no la llegue a usar nunca ya que la cache de `wexam-latex` evitará que se le pida compilar por segunda vez un mismo documento. En Amazon EC2 este último contenedor no está presente (no cabe) y en su lugar se usa un `latex-online` externo. Esto puede ser una brecha de seguridad ya que se comparte el fuente latex de un examen con el servidor externo.

Puedes ver los logs de los diferentes contenedores con `docker-compose logs`

## Setup inicial de la base de datos.

La base de datos comienza sin usuarios. Puedes crear el primero (que será el administrador) mediante el comando:

```
docker-compose exec wexam-server bash -c "RUN_ENV=default.yml \
                                 WEXAM_ADMIN_EMAIL=correo@del.admin.com \
                                 WEXAM_ADMIN_PASSWORD=laclavedeladmin \
                                 python seed_admin.py"
```

Una vez creado este primer usuario, pueden añadirse más con una herramienta de línea de comandos. Para ello ejecuta lo siguiente:

```
docker exec -it wexam-server bash
# Obtendrás un prompt dentro del contenedor wexam-server en él escribes lo siguiente
pip install requests
cd admin
python admin.py --help
```

El script `admin.py` te da ayuda sobre los parámetros que debes usar para conectar con el backend, básicamente se trata de `--user correo@del.admin.com`, `--password laclavedeladmin` y `--api https://wexam-server/wexam-api` por ejemplo. Si has dado los datos correctos, entras en un intérprete interactivo desde el que puedes listar usuarios, añadirlos, borrarlos o cambiarles la clave (pon `help` para ver los comandos).


## Compilación del cliente angular

El cliente angular está en la carpeta `../wexam-angular`, en código fuente. Para no depender del servidor de desarrollo (`ng serve`) es necesario compilar el cliente a una serie de archivos estáticos. De paso se puede "optimizar" la compilación para que ocupe lo menos posible y sea más rápida de descargar para el cliente.

Para esta compilación necesitaríamos `node` y `angular-cli`, pero usando la línea de tener todo en contenedores para no tener que instalar nada, se puede hacer uso de un contenedor público que trae estas herramientas preinstaladas y usarlo para compilar el cliente. Para ello:

* Ve a la carpeta `../wexam-angular`
* Ejecuta el comando

    ```bash
    docker run -u $(id -u) --rm -it -v "$PWD":/app trion/ng-cli:1.7.4 bash
    ```

Eso te abre un shell dentro de un contenedor donde está preinstalado angular-cli. Dentro de ese contenedor ejecuta:

```bash
npm install   # Este solo hace falta la primera vez, para instalar módulos npm
ng build      # Este cada vez que quieras rehacer la aplicación, durante desarrollo
ng build --prod   # Este para obtener la versión final "optimizada"
```

El resultado será una carpeta `dist` con la versión ya compilada. Ya que en ella está "hardcoded" el nombre de la API, es necesario modificarlo para quitar todo el prefijo `https://nombre.del.host/` y quedarse sólo con la parte `/wexam-api` ya que en este despliegue el cliente y el backend están alojados en el mismo servidor y por tanto no se requiere esa parte de la ruta. Este cambio puede hacerse con el siguiente comando:

```bash
sed -i 's/https:\/\/wexam.duckdns.org//g' main*js
```

El `docker-compose.yml` está configurado para montar la carpeta `dist` en un punto visible para `ngnix`, de modo que cuando éste reciba una petición a la raíz `/`, servirá la raíz de la carpeta `../wexam-angular/dist`. También está configurado para que, si recibe una petición a una ruta que no conozca, devuelva el archivo `index.html` en esa carpeta. Esto permite que si el cliente va, por ejemplo, a `https://nombre.del.nodo/problemas-list` (que es una de las vistas del cliente angular), funcione correctamente a pesar de que esa ruta no existe en el servidor.


# Uso

Una vez todo ha arrancado con éxito con `docker-compose up -d`, el Nginx estará escuchando por https en el puerto 8088 (puede cambiarse en el `docker-compose.yml`, de hecho en el `docker-compose.yml` para EC2 los puertos en que escucha son el 80 y el 443), y la API del backend estará escuchando por el protocolo wsgi-socket en el puerto 29000, pero el cliente conectará con el backend a través del mismo puerto de nginx (8088, 80 ó 443 depende), añadiendo la ruta `/wexam-api`, pues nginx redirigirá esa ruta al contenedor donde se ejecuta el backend.

Por tanto, basta dirigir un navegador a la máquina en que lo hayas desplegado, puerto 8088 (protocolo https) y la aplicación podrá usarse. 

En el despliegue para EC2, el navegador debe dirigirse a `https://wexam.duckdns.org` (nginx está configurado para redirigir a esta ruta https si el cliente usa por error http).

Haz login con alguno de los usuarios creados y puedes comenzar a crear preguntas y exámenes. Prueba a descargar uno como pdf.


[1]: https://docs.docker.com/install/
[2]: https://docs.docker.com/compose/install/
