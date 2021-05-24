# Ejercicio 3

Antes de entrar en los detalles de lo que se necesita para llevar un clúster a producción, su CTO desea que primero haga un alto para validar que su aplicación se pueda implementar en un entorno de Kubernetes.

## Objetivo

En este punto, ha creado las imágenes para los componentes de su aplicación y ha hecho que esas imágenes estén disponibles en su ACR privado.

El objetivo en este laboratorio es implementar su aplicación en un clúster de prueba de **Azure Kubernetes Service (AKS)** en su suscripción de Azure.

Asegurese de que todos sus contenedores estén activos y puedan comunicarse y llegar a los servicios de Azure necesarios. En particular:

- **TripViewer** debe poder acceder a los servicios de **Trips** y **UserProfile**

_**Nota**: Los enlaces de documentación de la API de Swagger en la página de inicio de tripviewer no funcionarán en este momento. Agregará esta funcionalidad en un laboratorio posterior._

- Las API de **POI** y de **User (Java)** deben ser accesibles (aunque la aplicación **Trip Viewer** no acceda a ellas en esta etapa). Consulte la documentación de la aplicación para conocer las formas de probar los puntos finales.

_**Nota**: No es necesario darle a cada API una IP externa._

Todas las API deben poder acceder a la base de datos SQL proporcionada en su suscripción de Azure. Los detalles de la conexión se lo pueden solicitar al Instructor. 

### Arquitectura Deseada

![Img5.png](/.attachments/Img5-80d40b00-75fe-4432-a48d-0012cdaad7f3.png)

## PASO 1 - Crear Cluster AKS

1. Cree un grupo de recursos con el comando **az group create**.

`az group create --name teamResources --location eastus`

2. Cree un clúster de AKS con el comando **az aks create**. En el siguiente ejemplo se crea un clúster denominado **myAKSCluster** con un nodo:

`az aks create --resource-group teamResources --name myAKSCluster --node-count 1 --enable-addons monitoring --generate-ssh-keys`

## PASO 2 - Conectarse al Cluster

Para administrar un clúster de Kubernetes, use **kubectl**, el cliente de línea de comandos de Kubernetes. Si usa Azure Cloud Shell, kubectl ya está instalado.

1. Instale kubectl localmente mediante el comando **az aks install-cli**:

`az aks install-cli`

2. Para configurar kubectl para conectarse a su clúster de Kubernetes, use el comando az aks get-credentials.

`az aks get-credentials --resource-group teamResources --name myAKSCluster`

3. Compruebe la conexión al clúster con el comando **kubectl get**. Este comando devuelve una lista de los nodos del clúster.

`kubectl get nodes`

La salida muestra el nodo único creado en los pasos anteriores. Asegúrese de que el estado del nodo es Listo:


| NAME | STATUS | ROLES | AGE | VERSION |
|--|--|--|--|--|
| aks-nodepool1-31718369-0 | Ready | agent | 6m44s | v1.19.9 |


## PASO 3 - Autenticar ACR desde AKS

Para integrar un ACR existente con clústeres de AKS existentes, proporcione valores válidos para **acr-name** o **acr-resource-id**, como se indica a continuación.

`az aks update -n myAKSCluster -g teamResources --attach-acr <acr-name>`

`az aks update -n myAKSCluster -g teamResources --attach-acr <acr-resource-id>`

## PASO 4 - Crear Namespace Kubernetes

Cree 2 Namespace (Api y Web) para desplegar tanto los componentes como la Aplicación Web

`kubectl create namespace <insert-namespace-name-here>`

## PASO 5 - Crear Archivo Configuración

1. Defina un Archivo de Configuración para almacenar las Credenciales de acceso a la BD.
Tenga en cuenta que el archivo de configuración se debe encontrar en cada Namespace donde se quiera utilizar

```
kubectl create secret generic sql -n <<namespace>> \
    --from-literal=SQL_USER=admindb \
    --from-literal=SQL_PASSWORD=Quality2021. \
    --from-literal=SQL_SERVER=qcsqlkube.database.windows.net \
    --from-literal=SQL_DBNAME=mydrivingDB
```
2. Cree un archivo llamado poi.yml y carguelo al Shell de Azure para desplegar nuestro servicio

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: poi-deployment
  namespace: api
  labels:
    deploy: poi
spec:
  replicas: 1
  selector:
    matchLabels:
      app: poi
  template:
    metadata:
      labels:
        app: poi
    spec:
      containers:
      - name: poi         
        image: "qcacrkube.azurecr.io/poi:1.0"
        imagePullPolicy: Always
        ports:
          - containerPort: 8080
            name: http
            protocol: TCP
          - containerPort: 443
            name: https
            protocol: TCP
        env:             
          - name: WEB_SERVER_BASE_URI
            value: 'http://0.0.0.0'
          - name: WEB_PORT
            value: '8080'
          - name: ASPNETCORE_ENVIRONMENT
            value: 'Development'
        envFrom:
          - secretRef:                                                                                                                                                                    
             name: sql

---
apiVersion: v1
kind: Service
metadata:
  name: poi
  namespace: api            
spec:
  selector:
    app: poi
  ports:
    - protocol: TCP
      name: poi-http
      port: 80
      targetPort: 8080
    - protocol: TCP
      name: poi-https
      port: 443
      targetPort: 443
```

3. Ejecute el archivo con el siguiente comando:

`kubectl apply -f poi.yml` 
y valide que la implementación halla finalizado exitosamente con la siguiente instrucción

`kubectl get deployments -n api`

4. Valide el Resultado

![Img7.png](/.attachments/Img7-dc206d15-3cd9-42f4-96ee-9fbc5a004c05.png)

5. Ejecute la siguiente instrucción para validar si el Microservicio **POI** tiene conectividad con la Base de Datos.

`kubectl exec <<pod_name>> -n <<namespace>> curl GET 'http://localhost:8080/api/poi'`

6. Deberá visualizar un resultado como el siguiente:

![Img6.png](/.attachments/Img6-82f85e18-7a16-4de6-8aff-eea3e60557dc.png)

7. Repita el procedimiento con los demás componentes (**user-java**, **userprofile** y **trips**)

#### Codigo Trips

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trips-deployment
  namespace: api
  labels:
    deploy: trips
spec:
  replicas: 1
  selector:
    matchLabels:
      app: trips
  template:
    metadata:
      labels:
        app: trips
    spec:
      containers:
      - name: trips
        image: qcacrkube.azurecr.io/trips:1.0
        imagePullPolicy: Always
        ports:
          - containerPort: 80
            name: http
            protocol: TCP
          - containerPort: 443
            name: https
            protocol: TCP
        env:
          - name: OPENAPI_DOCS_URI
            value: "http://localhost"
          - name: DEBUG_LOGGING
            value: "true"
          - name: PORT
            value: "80"
        envFrom:
          - secretRef:
              name: sql

---
apiVersion: v1
kind: Service
metadata:
  name: trips
  namespace: api
spec:
  type: ClusterIP
  selector:
    app: trips
  ports:
    - protocol: TCP
      name: trips-http
      port: 80
      targetPort: 80
    - protocol: TCP
      name: trips-https
      port: 443
      targetPort: 443  
```

#### Codigo UserProfile

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: userprofile-deployment
  namespace: api
  labels:
    deploy: userprofile
spec:
  replicas: 1
  selector:
    matchLabels:
      app: userprofile
  template:
    metadata:
      labels:
        app: userprofile
    spec:
      containers:
      - name: userprofile
        image: qcacrkube.azurecr.io/userprofile:1.0
        imagePullPolicy: Always        
        ports:
          - containerPort: 80
            name: http
            protocol: TCP
          - containerPort: 443
            name: https
            protocol: TCP
        envFrom:
          - secretRef:
              name: sql
              
---
apiVersion: v1
kind: Service
metadata:
  name: userprofile
  namespace: api
spec:
  type: ClusterIP
  selector:
    app: userprofile
  ports:
    - protocol: TCP
      name: userprofile-http
      port: 80
      targetPort: 80
    - protocol: TCP
      name: userprofile-https
      port: 443
      targetPort: 443

```

#### Codigo User-Java

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-java-deployment
  namespace: api
  labels:
    deploy: user-java
spec:
  replicas: 1
  selector:
    matchLabels:
      app: user-java
  template:
    metadata:
      labels:
        app: user-java
    spec:
      containers:
      - name: user-java
        image: qcacrkube.azurecr.io/user-java:1.0
        imagePullPolicy: Always
        ports:
          - containerPort: 80
            name: http
            protocol: TCP
          - containerPort: 443
            name: https
            protocol: TCP
        envFrom:
          - secretRef:
              name: sql
              
---
apiVersion: v1
kind: Service
metadata:
  name: user-java
  namespace: api
spec:
  type: ClusterIP
  selector:
    app: user-java
  ports:
    - protocol: TCP
      name: user-java-http
      port: 80
      targetPort: 80
    - protocol: TCP
      name: user-java-https
      port: 443
      targetPort: 443

```

#### Codigo TripViewer

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tripviewer-deployment
  namespace: web
  labels:
    deploy: tripviewer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tripviewer
  template:
    metadata:
      labels:
        app: tripviewer
    spec:
      containers:
      - name: tripviewer
        image: qcacrkube.azurecr.io/tripviewer:1.0
        imagePullPolicy: Always        
        ports:
          - containerPort: 80
            name: http
            protocol: TCP
          - containerPort: 443
            name: https
            protocol: TCP
        env:
          - name: USERPROFILE_API_ENDPOINT
            value: "http://10.0.48.61"
          - name: TRIPS_API_ENDPOINT
            value: "http://10.0.50.232"
          - name: BING_MAPS_KEY
            value: ""

---
apiVersion: v1
kind: Service
metadata:
  name: tripviewer
  namespace: web
spec:
  type: LoadBalancer
  selector:
    app: tripviewer
  ports:
    - protocol: TCP
      name: tripviewer-http
      port: 80
      targetPort: 80
    - protocol: TCP
      name: tripviewer-https
      port: 443
      targetPort: 443

```

8. Ejecute la siguiente instrucción para visualizar la IP Publica de tripviewer

`kubectl get services -n web`

9. Desde un Navegador acceda a la IP Publica, valide el acceso a la Aplicación Web y luego ingrese a cada menu de la parte superior para garantizar el acceso a las APIs **Userprofile** y **Trips**

![Img8.png](/.attachments/Img8-abcaebe5-bd1f-4dd5-8430-f66ed1dec1d3.png)

---

Buen Trabajo :)
