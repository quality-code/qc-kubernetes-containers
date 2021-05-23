# Ejercicio 2 - Azure Container Registry

Ahora que está seguro de que la aplicación de **POI** funciona, debe asegurarse de que todos los componentes de **TripInsights** se compilen como imágenes de Docker y se envíen al Azure Container Registry (ACR) que ha creado.

Si no ha creado el ACR puedes obtener una guia en:

https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-portal

Consulte el recurso de Azure Container Registry en Azure Portal para obtener las credenciales del registro.

![Img4.png](https://github.com/quality-code/qc-kubernetes-containers/blob/master/Labs/Resources/Img4.png)

## PASO 1

Ajuste el archivo **Dockerfile** para cada uno de los Microservicios y dejelos dentro del directorio de cada proyecto.

Compile cada una de las imagenes

#### TRIPS

`docker image build -t qualitycode1/trips:1.0 .`

```
docker run -d \
    --network qcVnet \
    -p 8081:80 \
    --name trips \
    -e "SQL_PASSWORD=Quality123." \
    -e "SQL_SERVER=sqltestdb" \
    -e "SQL_USER=sa" \
    -e "OPENAPI_DOCS_URI=http://temp" \
    qualitycode1/trips:1.0
```

#### USER-JAVA

`docker image build -t qualitycode1/user-java:1.0 .`

```
docker run -d \
    --network qcVnet \
    -p 8082:80 \
    --name user-java \
    -e "SQL_PASSWORD=Quality123." \
    -e "SQL_SERVER=sqltestdb" \
    -e "SQL_USER=sa" \
    qualitycode1/user-java:1.0
```

#### USERPROFILE

`docker image build -t qualitycode1/userprofile:1.0 .`

```
docker run -d \
    --network qcVnet \
    -p 8083:80 \
    --name userprofile \
    -e "SQL_PASSWORD=Quality123." \
    -e "SQL_SERVER=sqltestdb" \
    -e "SQL_USER=sa" \
    qualitycode1/userprofile:1.0
```

Si elige testear el resto de las imágenes, puede ejecutarlas localmente y enviar una solicitud **HTTP GET** al endpoint **healtcheck**. Por ejemplo, para acceder al endpoint **healtcheck** de POI en un contenedor que se ejecuta localmente en el puerto 8080, ejecute un **curl** o visite **http://localhost:8080/api/poi/healthcheck** en el navegador.

Es posible que los **Endpoints** distintos del endpoint **healtcheck** no sean funcionales en este momento (debido a las dependencias de las API o la base de datos SQL), así que no se preocupe si no puede llegar a ellos.

#### ACR

Cargue las Imagenes al ACR

___

### Buen Trabajo :)
