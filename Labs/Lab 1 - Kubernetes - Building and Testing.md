# Ejercicio 1 - Building and Testing

## Contenedores

Los contenedores se han adoptado como una excelente manera de aliviar los problemas de portabilidad. Forman el núcleo de este Laboratorio y sustentan todo lo que explorarás a medida que avanzas en los Ejercicios.

El objetivo de este Ejercicio es asegurarse de que comprende los conceptos básicos de los contenedores, puede trabajar con ellos localmente y enviarlos a un repositorio de imágenes de contenedores.

## Objetivo

Se le asignó la tarea de mejorar la experiencia de desarrollo local para los nuevos desarrolladores mediante el uso de Docker para simplificar la construcción, las pruebas y la ejecución de la aplicación. 

Al CTO también le gustaría que esto se convierta en parte de la solución de prueba de integración del proceso de compilación.

Se ha realizado parte del trabajo por usted, pero fue durante una época en la que los equipos se dividían entre operaciones y desarrollo, dejando el código dividido entre varias bases de código. 

El nuevo CTO cree que los equipos deberían ser una mezcla de Operaciones y Desarrollo y ha formado el equipo en el que estás ahora.

## Building and Testing

Como eres nuevo en esta base de código, vas a verificar que al menos uno de los servicios aún funciona mediante la compilación y la prueba local.

Para hacer esto, deberá crear y ejecutar el contenedor de **Points of Interest (POI)**, así como un contenedor de **SQL Server**. El contenedor de _Points of Interest_ se comunica con el contenedor de _SQL Server_ a través de la red de Docker:

![Img2.png](https://github.com/quality-code/qc-kubernetes-containers/blob/master/Labs/Resources/Img2.png)

Para crear la aplicación de **POI**, use el **[código fuente de TripInsights](https://github.com/quality-code/qc-kubernetes-containers)** y Dockerfile para cada microservicio, haciendo coincidir el Dockerfile con el código fuente.

_**Sugerencia**_: _si tiene problemas para hacer coincidir el Dockerfile con el código fuente, recuerde que los servicios están escritos en diferentes lenguajes. Eche un vistazo dentro del Dockerfile, el servicio correspondiente es más obvio de lo que cree._

### PASO 1

Cree una Red en Docker llamada **qcVnet**

`docker network create qcVnet`

### PASO 2

Ejecute un contenedor con la última versión de MS-SQL Server para desplegar la Base de Datos con la cual testearemos la Aplicación **POI**

```
docker run -d \
    --network qcVnet \
    -e 'ACCEPT_EULA=Y' \
    -e 'SA_PASSWORD=Quality123.' \
    --name 'sqltestdb' \
    -p 1433:1433 \
    mcr.microsoft.com/mssql/server:2019-latest
```

### PASO 3

Inserte la información de Prueba dentro de la Base de Datos del Contenedor **sqltestdb**

```
docker run -d \
    --network qcVnet \
    --name dataload \
    -e "SQLFQDN=sqltestdb" \
    -e "SQLUSER=sa" \
    -e "SQLPASS=Quality123." \
    -e "SQLDB=mydrivingDB" \
    qualitycode1/data-load:v1
```
### PASO 4

Copie el **Dockerfile** dentro de la carpeta del proyecto **POI** para generar la imagen por medio del siguiente comando:

`docker image build -t qualitycode1/poi:1.0 .`

### PASO 5
Ahora que ha instalado, configurado y poblado la Base de Datos, debe ejecutar el Contenedor de **Points of Interest API** con la siguiente instrucción


```
docker run -d \
    --network qcVnet \
    -p 8080:80 \
    --name poi \
    -e "SQL_PASSWORD=Quality123." \
    -e "SQL_SERVER=sqltestdb" \
    -e "SQL_USER=sa" \
    -e "ASPNETCORE_ENVIRONMENT=Local" \
    qualitycode1/poi:1.0
```


### PASO 6

Desde un navegador Ingrese a http://localhost:8080/api/poi para validar la conectividad de la aplicación POI hacia la Base de Datos.

![Img3.png](https://github.com/quality-code/qc-kubernetes-containers/blob/master/Labs/Resources/Img3.png)

___

### BUEN TRABAJO :)
