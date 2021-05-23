
# Escenario

Uno de sus productos brinda a los clientes la oportunidad de calificar para tarifas de seguro de automóvil más bajas.

Los clientes pueden hacerlo si optan por utilizar la aplicación **TripInsights**, que recopila datos sobre sus hábitos de conducción.

Se ha asignado a su equipo la tarea de modernizar la aplicación y trasladarla a la nube.

La aplicación **TripInsights**, que alguna vez fue un monolito, se ha refactorizado en varios microservicios:

![Img1.png](https://github.com/quality-code/qc-kubernetes-containers/blob/master/Labs/Resources/Img1.png)

- **Trip Viewer WebApp (.NET Core)**: sus clientes utilizan esta aplicación web para revisar sus puntajes de manejo y viajes. Los viajes se están simulando contra las API dentro del entorno del Laboratorio.
- **Trip API (Go)**: la aplicación móvil envía los datos de viaje de diagnóstico a bordo (OBD) del vehículo a esta API para que se almacenen.
- **Points of Interest API (.NET Core)**: esta API se utiliza para recopilar los puntos del viaje cuando se detecta una parada brusca o una aceleración brusca.
- **User Profile API (NodeJS)**: la aplicación utiliza esta API para leer la información del perfil del usuario.
- **User API (Java)**: la aplicación utiliza esta API para crear y modificar los usuarios.

