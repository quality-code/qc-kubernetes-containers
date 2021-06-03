# Escenario

Ahora que su clúster está conectado a su red de producción y ha pasado la revisión inicial del equipo de seguridad, es hora de implementar la aplicación TripInsights en el clúster configurado correctamente.

## Objetivo

El objetivo en este laboratorio es implementar todos los contenedores en su clúster. Seguiremos agregando características para mejorar la seguridad de las Aplicaciones.

## Seguridad

A medida que implementa sus servicios en su clúster, tenga en cuenta los requisitos de seguridad exigidos por su CTO:

1. En última instancia, habrá varios equipos trabajando en el clúster: los equipos de administración, desarrollo web y desarrollo de API. Las aplicaciones web y API deben implementarse en espacios de nombres separados.

2. Bloquee el acceso de los usuarios a tu clúster. Si bien es parte del equipo de administración y tiene permisos para acceder a cualquier recurso en el clúster, hay otros dos usuarios de los equipos web y API en su AAD que solo deberían tener acceso a los espacios de nombres apropiados:

- usuario **aksweb** (Acceso **View** para recursos de API y acceso **Edit** para recursos web).
- usuario **aksapi** (Acceso **View** para recursos web, Acceso **Edit** para recursos API)

3. Los **secretos** deben protegerse en una _bóveda externa_, no en el clúster. Este enfoque evita que cualquier persona sin permisos o acceso a la bóveda acceda directamente a los valores.

## Ingress

Aunque tiene varios servicios implementados en el clúster, sería bueno tener un único **Endpoint**. Para hacer esto, cree un **Ingress Controller** y configure las reglas de ingreso para enrutar a los servicios apropiados.
Puede realizar llamadas a las API por su nombre (http://endpoint.you.provide/api/trips).

## Criterios de Aceptación

- Implemente con éxito la aplicación TripInsights en el clúster.

- Debe poder conectarse al clúster utilizando los usuarios de AAD **aksapi** y **aksweb** y demostrar los niveles de acceso adecuados.

- Asegure la información de conexión de **Azure SQL Server** de manera que no se pueda acceder a valores literales de manera inapropiada.

- Use una bóveda de claves externa para almacenar y acceder a secretos dentro de su clúster

- Asegúrese de que se pueda acceder a todos los enlaces del sitio de Trip Viewer.

# Implementar la Aplicación

## PASO 1 - Preparar Entorno

1. Asegúrese de Crear los namespaces **web** y **api** dentro del Cluster creado en el Laboratorio Anterior. Puede tomar como guia lo realizado en el Laboratorio 3.

2. Despliegue cada uno de los **API Services** y **Tripviewer** dentro del AKS Cluster.

3. Realice un Test a sus APIs para garantizar que funcionen correctamente.

# Acceso e identidad en Azure Kubernetes Service (AKS)

## PASO 2 - Acceso e identidad en Azure Kubernetes Service (AKS)

1. Recupere el ID del AKS Cluster para luego vincularlos a los Grupos que creará.

```
AKS_ID=$(az aks show \
    --resource-group teamResources \
    --name myAKSCluster \
    --query id -o tsv)
```

2. Cree un grupo AAD llamado **webdev** para asignarlo mas adelante al namespace **web**:

```
WEBDEV_GROUP_ID=$(az ad group create \
    --display-name Web-dev \
    --mail-nickname webdev \
    --query objectId -o tsv)
```

3. Cree un grupo AAD llamado **apidev** para asignarlo mas adelante al namespace **api**:

```
APIDEV_GROUP_ID=$(az ad group create \
    --display-name Api-dev \
    --mail-nickname apidev \
    --query objectId -o tsv)
```

4. Obtenga y guarde los ID de cada grupo para utilizarlos mas adelante.

```
echo $WEBDEV_GROUP_ID
echo $APIDEV_GROUP_ID
```

5. Realice la asignación del Role Azure **Kubernetes Service Cluster User Role** para cada uno de los grupos creados anteriormente.

```
az role assignment create \
    --assignee $WEBDEV_GROUP_ID \
    --role "Azure Kubernetes Service Cluster User Role" \
    --scope $AKS_ID
```

```
az role assignment create \
    --assignee $APIDEV_GROUP_ID \
    --role "Azure Kubernetes Service Cluster User Role" \
    --scope $AKS_ID
```

## PASO 3 - Crear Usuarios AAD

Cree dos usuarios de prueba dentro del AAD (**aksapi** y **aksweb**) y asígnelos a su respectivo grupo (**apidev** y **webdev**)

```
AKSDEV_ID=$(az ad user create \
  --display-name "AKS Api" \
  --user-principal-name aksapi@qualitycode.com.co \
  --password Quality2021. \
  --query objectId -o tsv)
```

`az ad group member add --group Api-dev --member-id $AKSDEV_ID`

```
AKSDEV_ID=$(az ad user create \
  --display-name "AKS Web" \
  --user-principal-name aksweb@qualitycode.com.co \
  --password Quality2021. \
  --query objectId -o tsv)
```

`az ad group member add --group Web-dev --member-id $AKSDEV_ID`

## PASO 4 - Crear un RoleBinding

Cree un archivo YAML con el nombre **api-group-role-binding.yaml** y agregue el siguiente script:

Este Script se encarga de asignar permisos de **edicion** sobre el **namespace api** y permisos de **solo lectura** sobre el **namespace web**

Reemplace _<<GroupID AAD - ApiDev>>_ con el ID del Grupo ApiDev que genero anteriormente.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: api
  name: api-dev-rolebinding
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: Group
  name: <<GroupID AAD - ApiDev>>
  apiGroup: rbac.authorization.k8s.io
  namespace: api
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: web
  name: api-dev-rolebinding
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: Group
  name: <<GroupID AAD - ApiDev>>
  apiGroup: rbac.authorization.k8s.io
  namespace: web
```

Cree un archivo YAML con el nombre **web-group-role-binding.yaml** y agregue el siguiente script:

Este Script se encarga de asignar permisos de **edicion** sobre el **namespace web** y permisos de **solo lectura** sobre el **namespace api**

Reemplace _<<GroupID AAD - WebDev>>_ con el ID del Grupo WebDev que genero anteriormente.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: web
  name: web-dev-rolebinding
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: Group
  name: <<GroupID AAD - WebDev>>
  apiGroup: rbac.authorization.k8s.io
  namespace: web
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: api
  name: web-dev-rolebinding
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: Group
  name: <<GroupID AAD - WebDev>>
  apiGroup: rbac.authorization.k8s.io
  namespace: api
```

## PASO 5 - Testear el Acceso al Cluster AKS

1. Cambie el contexto de ejecución del Cluster AKS para validar el acceso con alguno de los usuarios creados anteriormente.

`az aks get-credentials --resource-group teamResources --name myAKSCluster --overwrite-existing`

2. Ejecute la siguiente instrucción para que AKS solicite el inicio de sesión.

`kubectl get pods --namespace api`

3. Complete el proceso de Inicio de Sesión con el usuario **ApiDev** y Ejecute el siguiente comando para validar que pueda ejecutar **PODS** dentro del namespace.

`kubectl run nginx-dev --image=nginx --namespace api`

4. Trate de ejecutar la instrucción anterior dentro del namespaces **web** y notara que genera un error por falta de permisos.

5. Repita los puntos 1 al 4 para iniciar sesión con el usuario **WebDev** y valide que no  pueda realizar creación y/o modificación de elementos dentro del namespaces **api**

# Azure Key Vault Provider for Secrets Store CSI Driver

URLs de Referencia:
- https://github.com/Azure/secrets-store-csi-driver-provider-azure
- https://azure.github.io/secrets-store-csi-driver-provider-azure/getting-started/installation/
- https://secrets-store-csi-driver.sigs.k8s.io/getting-started/installation.html


## PASO 6 - Asegurar información con Azure Key Vault

1. Cree un nuevo **Azure Key Vault** llamado **qcAKV**

`az keyvault create --name qcAKV --resource-group "teamResources" --location "EastUS"`

2. Ingrese al Key Vault y agregue los Secrets necesarios para acceder a la Base de Datos SQL Server.


| SECRET | VALUE |
|--|--|
| SQL-USER | admindb |
| SQL-PASSWORD | Quality2021. |
| SQL-SERVER | qcsqlkube.database.windows.net |
| SQL-DBNAME | mydrivingDB |

3. Cree un ServicePrincipal para asociarlo al **AKS Cluster** y posteriomente otorgarle permisos sobre el **Azure Key Vault**.

`az ad sp create-for-rbac --skip-assignment --name qcAKSClusterSP`

Copie el Resultado, el cual sera como el siguiente:

```
{
  "appId": "8bccaa3a-3ee4-43a4-91ec-fbb91e159b8f",
  "displayName": "qcAKSClusterSP",
  "name": "http://qcAKSClusterSP",
  "password": "VYXP1LyIVgCINmnY~ZQ_SqewzZDATMJ7eI",
  "tenant": "353a1d98-d4cd-4905-be46-9388f4f011a1"
}
```

4. Cree un **Secret** dentro de Kubernetes para almacenar los datos del **ServicePrincipal** creado en el paso anterior.

`kubectl create secret generic secrets-store-creds --from-literal clientid=<<appId>> --from-literal clientsecret=<<password>>  -n api`

5. Asigne Permisos de Lectura al **ServicePrincipal** sobre los _Secrets_, _Keys_ y _Certificates_ dentro del **Azure Key Vault**.

```
SPNAME=qcAKSClusterSP
AZURE_CLIENT_ID=$(az ad sp show --id http://${SPNAME} --query appId -o tsv)
KEYVAULT_NAME=qcAKV
```

```
az keyvault set-policy -n $KEYVAULT_NAME --key-permissions get --spn $AZURE_CLIENT_ID
az keyvault set-policy -n $KEYVAULT_NAME --secret-permissions get --spn $AZURE_CLIENT_ID
az keyvault set-policy -n $KEYVAULT_NAME --certificate-permissions get --spn $AZURE_CLIENT_ID
```

6. Para realizar una administración de Contraseñas, Llaves y Certificados de forma segura, se debe implementar un Driver para la Administración de Secretos (**Secrets Store CSI Driver**), ésto se lograra con las siguientes instrucciones:

```
helm repo add secrets-store-csi-driver https://raw.githubusercontent.com/kubernetes-sigs/secrets-store-csi-driver/master/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system
```

7. Una vez finalizada la instrucción, verifique que el Driver haya iniciado correctamente:

`kubectl --namespace=kube-system get pods -l "app=secrets-store-csi-driver"`

notara un resultado como el siguiente:

![Img15.png](/.attachments/Img15-49ae2afe-be25-41b1-b056-28db00b771e2.png)

8. En vista de que se requiere integrar una bóveda externa, se ha seleccionado Azure Key Vault para cumplir este requerimiento, por lo tanto debe instalar el **Azure Key Vault provider** por medio del siguiente comando:

`kubectl apply -f https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/deployment/provider-azure-installer.yaml`

9. Con la siguiente instrucción valide que el **Azure Key Vault provider** se esté ejecutando como se espera

`kubectl get pods -l app=csi-secrets-store-provider-azure`

![Img16.png](/.attachments/Img16-3605c56d-a9aa-4900-a3cb-647fbdd3f612.png)

# Proporcione la Identidad para Acceder al Key Vault

## PASO 1 - Ejecutar Archivos YAML para conectarse al AKV

Cree un recurso personalizado **SecretProviderClass** para proporcionar parámetros específicos del proveedor para el _Secrets Store CSI driver_. 

1. Cree un archivo YAML con el nombre _SecretProviderClass.yaml_ y agregue el siguiente contenido para asociar los secrets configurados dentro de las APIs de la aplicación con los Secrets configurados en el AKV:

NOTA: _Cambie el Tenant ID por el generado en el punto 2 del paso 6._

```
# This is a SecretProviderClass example using a service principal to access Keyvault
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: azure-kvname
  namespace: api
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"         # [OPTIONAL] if not provided, will default to "false"
    keyvaultName: "qcAKV>"          # the name of the KeyVault
    cloudName: ""                   # [OPTIONAL for Azure] if not provided, azure environment will default to AzurePublicCloud
    objects:  |
      array:
        - |
          objectName: SQL-USER
          objectType: secret
          objectAlias: SQL_USER        
          objectVersion: ""         
        - |
          objectName: SQL-PASSWORD
          objectType: secret
          objectAlias: SQL_PASSWORD
          objectVersion: ""
        - |
          objectName: SQL-SERVER
          objectType: secret
          objectAlias: SQL_SERVER
          objectVersion: ""
        - |
          objectName: SQL-DBNAME
          objectType: secret
          objectAlias: SQL_DBNAME
          objectVersion: ""
    tenantId: "<<YourTenantID>>"                 # the tenant ID of the KeyVault
```

2. Aplique el archivo _SecretProviderClass.yaml_ dentro del Cluster AKS.

`kubectl apply -f SecretProviderClass.yaml`

## PASO 2 - Actualice sus Despliegues YAML

Para asegurarse de que su aplicación esté usando el **Secrets Store CSI driver**, actualice su **yaml** de implementación para usar el driver **secrets-store.csi.k8s.io** y haga referencia al recurso **SecretProviderClass** creado en el paso anterior. 

1. Cree un Volume dentro de cada archivo de implementación YAML.

      
```
      volumes:
      - name: secrets-store-inline
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "azure-kvname"
          nodePublishSecretRef:                       
            name: secrets-store-creds
```

2. Una vez creado el Volume, debe montarlo dentro de las especificaciones del contenedor con la siguiente configuración:

        
```
        volumeMounts:
          - name: secrets-store-inline
            mountPath: "/secrets"
            readOnly: true
```

3. Luego de Modificar nuestro archivo YAML para el **API poi.yml** se visualizara de la siguiente manera:

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
          - containerPort: 80
            name: http
            protocol: TCP
          - containerPort: 443
            name: https
            protocol: TCP
        env:             
          - name: WEB_SERVER_BASE_URI
            value: 'http://0.0.0.0'
          - name: WEB_PORT
            value: '80'
          - name: ASPNETCORE_ENVIRONMENT
            value: 'Development'
        volumeMounts:
          - name: secrets-store-inline
            mountPath: "/secrets"
            readOnly: true
      volumes:
      - name: secrets-store-inline
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "azure-kvname"
          nodePublishSecretRef:                       # Only required when using service principal mode
            name: secrets-store-creds                 # Only required when using service principal mode. The name of the Kubernetes secret that contains the service principal credentials to access keyvault.

---
apiVersion: v1
kind: Service
metadata:
  name: poi
  namespace: api            
spec:
  type: ClusterIP
  selector:
    app: poi
  ports:
    - protocol: TCP
      name: poi-http
      port: 80
      targetPort: 80
    - protocol: TCP
      name: poi-https
      port: 443
      targetPort: 443
```

4. Realice la modificación del **Volume** para **cada API** de nuestra aplicación **Tripviewer**.

5. Aplique nuevamente cada archivo YAML de las APIs sobre el Cluster AKS.

```
kubectl apply -f poi.yml
kubectl apply -f trips.yml
kubectl apply -f userprofile.yml
kubectl apply -f user-java.yml
```

6. Consulte el POD del **API POI** para testear que pueda conectarse a la Base de Datos y recuperar información:

`kubectl get pods -n api -l app=poi`

![Img17.png](/.attachments/Img17-853bf8c5-8e43-4cdd-be75-264a9afac0db.png)

7. Tome el nombre del POD y ejecute la siguiente instrucción para testear el API.

`kubectl exec poi-deployment-6454ff55c4-xrsxp -n api -- ls ../secrets`

El resultado mostrará los Secrets recuperados desde el **Azure Key Vault**.

![Img18.png](/.attachments/Img18-98bf992a-510d-4103-ba61-30d370a99a3e.png)

8. Si desea visualizar el valor de alguno de estos secrets, ejecute la siguiente instrucción dentro del POD:

![Img19.png](/.attachments/Img19-7f303755-b053-45e3-bddb-0cd1d87e4382.png)

# Buen Trabajo :) 
