# Escenario

Su CTO quedó impresionado con su capacidad para demostrar que AKS puede respaldar fácilmente su aplicación mediante una implementación de prueba. 

Sin embargo, estuvo de acuerdo en que esta implementación no sería aprobada por su equipo de seguridad interno ni cumpliría con los requisitos de auditoría.

En un proximo laboratorio, deberá configurar un clúster que, en última instancia, se convertirá en parte de la infraestructura en la nube existente de la Compañía.

## Objetivo

El objetivo en este laboratorio es crear y configurar un clúster de Kubernetes en Azure con las medidas de seguridad adecuadas. Su empresa trata con información confidencial, por lo que es imperativo que aborde la seguridad al configurar su clúster. 

- Debe integrarse con el tenant de Azure Active Directory (AAD) de su empresa para implementar el control de acceso basado en roles (RBAC) para la autenticación del clúster.
- Proteger sus recursos mediante el uso de una red virtual dedicada y proteger la parte más crítica de su clúster de Kubernetes, la API de Kubernetes.

## Configuración

A medida que configura su clúster, su CTO desea que considere lo siguiente:

1. Debido al tamaño de la empresa, se están utilizando muchos de los espacios de direcciones IP privados. Aprovisione una **VNet** con un rango de IP para ejecutar aplicaciones dentro de Azure. Cree y asocie la **SubNet** al Cluster de Kubernetes.

2. Los usuarios de **TripInsights** esperan que sus datos sean precisos y estén actualizados en todo momento. Es importante considerar la disponibilidad de la aplicación para informar su decisión sobre la cantidad de nodos en su clúster.

3. Solo los miembros de su equipo deberían poder acceder al servidor de Kubernetes.

4. Los pods de su clúster deberían poder comunicarse directamente con otros recursos en la **VNET** a través de direcciones IP privadas.

## Uso de RBAC (control de acceso basado en roles)

RBAC se utiliza para asignar roles (un grupo de permisos a los recursos) a los usuarios (cualquier entidad que accede a un recurso de forma interactiva) o cuentas de servicio (cualquier entidad que accede a un recurso de forma no interactiva e independiente de un usuario).

El uso de estas construcciones le permite separar los permisos entre diferentes usuarios y participar en el principio de privilegio mínimo. Este principio sugiere que a cualquier usuario o cuenta de servicio se le deben asignar roles con el privilegio mínimo necesario para acceder a los recursos que necesitan para completar su rol operativo en el clúster y para cada aplicación.

## Protección de recursos con una red virtual

Al igual que con muchos otros servicios de Azure, puede proteger sus nodos de Kubernetes colocándolos en una red virtual. El uso de una red virtual evita conexiones externas no autorizadas y puede aumentar la seguridad de los servicios administrados correspondientes.

## Criterios de Aceptación

- Crea con éxito un **clúster AKS habilitado para RBAC** dentro del espacio de direcciones que creó para la **SubNet**.

- Debe demostrar que se le solicita el acceso al clúster para autenticarse con AAD

- Debe demostrar la conectividad hacia y desde su clúster al poder acceder a la máquina virtual interna (ya implementada)

## PASO 1 - Crear Grupo AAD

`az ad group create --display-name qcAKSAdminGroup --mail-nickname qcAKSAdminGroup`

![Img9.png](https://github.com/quality-code/qc-kubernetes-containers/blob/master/Labs/Resources/Img9.png)

Copie el valor de **objectId** para utilizarlo en un paso posterior.

## PASO 2 - CREAR VNet y SubNet

Ejecute la siguiente instrucción para declarar las variables a utilizar en los proximos comandos.

```
rg=teamResources
vnet=vnet
addressPrefix=10.240.1.0/24
clusterName=myAKSCluster
```

La siguiente instrucción creará una VNet y una SubNet asociada con un segmento de red sobre el cual se crearán los PODS dentro de Kubernetes.

`az network vnet create -g $rg --name $vnet --address-prefixes 10.0.0.0/8 -o none` 
`az network vnet subnet create -g $rg --vnet-name $vnet --name vm-subnet --address-prefixes $addressPrefix -o none`

Ejecute el siguiente comando para obtener la lista de SubNets para recuperar el Id de la SubNet que deseamos asociar al Cluster de Kubernetes.

`az network vnet subnet list -g $rg--vnet-name $vnet`

![Img10.png](https://github.com/quality-code/qc-kubernetes-containers/blob/master/Labs/Resources/Img10.png)

## PASO 3 - Crear un Cluster AKS-managed Azure AD

La siguiente instrucción crea un Cluster AKS integrado con Azure Active Directory (AAD).

Reemplace los valores para **<<SubNetID>>** y **<<ObjectID>>** con los valores generados anteriormente.

```
az aks create \
    --resource-group $rg \
    --name $clusterName \
    --node-count 1 \
    --network-plugin azure \
    --vnet-subnet-id <<SubNetID>> \
    --docker-bridge-address 172.17.0.1/16 \
    --dns-service-ip 10.2.1.10 \
    --service-cidr 10.2.1.0/24 \
    --generate-ssh-keys \
    --enable-aad \
    --aad-admin-group-object-ids <<ObjectID>>
```

## PASO 4 - Obtener Credenciales como Administrator para el AKS

Dado que el AKS fue creado en integración para el AAD, no podra interactuar contra el Cluster hasta tanto no halla asignado el Role **rbac.authorization.k8s.io** al usuario designado por la compañia, obtendremos las credenciales en modo Administrador para definir la configuración adicional, esto lo lograra con el atributo **--admin**

`az aks get-credentials --resource-group $rg --name $clusterName --overwrite-existing --admin`

Vincularemos al Container Registry Creado en un Laboratorio anterior al Cluster creado en este Laboratorio y de esta manera poder acceder a las imagenes que se encuentren allí disponibles.

`az aks update -n $clusterName -g $rg --attach-acr qcacrkube`

## PASO 5 - Crear YAML con la definición para asignar Role AKS

1. Cree un archivo llamado **basic-azure-ad-binding.yaml** y agregue el siguiente Script:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: qc-cluster-admins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: <<Email Corporativo>>
```

2. Ejectuar RBAC para que el usuario corporativo pueda acceder al Cluster en modo Administrador desde el Portal

`kubectl apply -f basic-azure-ad-binding.yaml`

3. Sobreescriba las credenciales de Acceso al AKS para que al ejecutar algun comando, solicite las credenciales de acceso contra el AAD.

`az aks get-credentials --resource-group $rg --name $clusterName --overwrite-existing`

4. Ejecute la siguiente instrucción para recuperar los nodos dentro del Cluster AKS.

`kubectl get nodes`

5. Recibira un Mensaje solicitando iniciar sesión con su cuenta del AAD.

![Img11.png](https://github.com/quality-code/qc-kubernetes-containers/blob/master/Labs/Resources/Img11.png)

6. Abra la URL e ingrese el codigo de confirmación para iniciar sesión.

![Img12.png](https://github.com/quality-code/qc-kubernetes-containers/blob/master/Labs/Resources/Img12.png)

7. Al Iniciar Sesión correctamente, se le mostrará una imagen como la siguiente:

![Img13.png](https://github.com/quality-code/qc-kubernetes-containers/blob/master/Labs/Resources/Img13.png)

8. Regrese a la Ventana del Shell y observará el resultado de la Instrucción:

![Img14.png](https://github.com/quality-code/qc-kubernetes-containers/blob/master/Labs/Resources/Img14.png)

## PASO 6 - Despliegue aplicación POI para validar conectividad

1. Cree un Namespaces llamado **api**

`kubectl create namespace api`

2. Despligue el Servicio de **POI**

`kubectl apply -f poi.yaml`

3. Consulte los PODS disponibles y copie el nombre del POD.

`kubectl get pod -n api`

4. Puede consultar la **IP del POD** con la siguiente instrucción y observe que corresponde al segmento definido dentro de la **SubNet**:

`kubectl get pod -n api -o wide`

5. Conectese interactivamente al POD para validar su funcionalidad

`kubectl exec -n api -it <<PodName>> -- sh`

5. Dentro del Shell del POD, ejecute el siguiente Script para validar su funcionamiento y conectividad a la Base de Datos.

`curl -i -X GET 'http://localhost:8080/api/poi/healthcheck'`

6. Para complementar este ejercicio podria crear una **Máquina Virtual** asociada a la **VNet** creada anteriormente y acceder desde esta maquina a la URL del **servicio POI** o hacer **PING** y notara que puede visualizar los datos sin problema.

# Buen Trabajo :)
