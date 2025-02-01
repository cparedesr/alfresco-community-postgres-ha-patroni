# Alfresco Community Con Postgres HA 
Este repositorio se basa en la necesidad de habilitar Postgresql en modo high availability con patroni y zookeeper. Como aplicación, hemos conectado Alfresco Community Edition v23.4 y realizado diferentes pruebas para garantizar la integridad de los datos.

## Arquitectura Postgresql + Patroni + Zookeeper + HaProxy
A continuación, mostramos la arquitectura respresentada en la colección de docker que se encuentra en la carpeta postgres-hs-patroni.
```
                         +------------------------------+
                         |          HAProxy             |
                         |------------------------------|
                         | - Balanceador de carga       |
                         | - Verifica el estado de      |
                         |   pg-master y pg-slave       |
                         | - Puertos expuestos:         |
                         |   • 5432 (tráfico PostgreSQL)|
                         |   • 7000 (estadísticas)      |
                         +--------------+---------------+
                                        |
                                        | (Conexiones TCP entrantes)
                                        V
               +------------------------+------------------------+
               |                                                 |
       +---------------+                                  +---------------+
       |  pg-master    |                                  |   pg-slave    |
       | (PostgreSQL + |                                  | (PostgreSQL + |
       |   Patroni)    |                                  |   Patroni)    |
       | - API REST    |                                  | - API REST    |
       |   en 8008     |                                  |   en 8008     |
       | - Volumen de  |                                  | - Volumen de  |
       |   datos propio|                                  |   datos propio|
       +-------+-------+                                  +-------+-------+
               |                                                 |
               |                                                 |
               +----------------------+--------------------------+
                                      |
                                      | (Coordinación y estado)
                                      V
                              +-----------------------+
                              |  Zookeeper            |
                              |-----------------------|
                              | - Servicio de         |
                              |   coordinación        |
                              | - Elección de         |
                              |   líder (failover)    |
                              | - Comunicación con    |
                              |   Patroni en cada nodo|
                              +-----------------------+
```
## HaProxy
Función: Actúa como balanceador de carga que recibe las conexiones entrantes en el puerto 5432 (destinado a PostgreSQL) y distribuye el tráfico a los nodos disponibles (pg-master y pg-slave).
Verificación: Realiza comprobaciones de estado (health-check) a través de las APIs REST expuestas en el puerto 8008 de cada nodo para saber cuál es el maestro activo y si los esclavos están en condiciones de recibir conexiones.

## PostgreSQL + Patroni
Función: Son los nodos que componen el clúster de PostgreSQL. Cada uno se ejecuta con Patroni, que se encarga de:
- Sincronización y replicación: El nodo maestro atiende las escrituras, mientras que el nodo esclavo se mantiene replicado y listo para asumir en caso de fallo del maestro.
- API REST en el puerto 8008: Permite a HAProxy y a otros componentes (como Patroni) monitorear el estado de cada nodo.

Almacenamiento: Cada uno debe utilizar un volumen de datos independiente para que tengan su propia copia local y evitar conflictos.

## Zookeeper
Función: Es el servicio centralizado de coordinación que utiliza Patroni para:
- Elección de líder: En caso de fallo del nodo maestro, ayuda a coordinar la promoción de uno de los nodos esclavos a maestro.
- Sincronización de configuraciones y estado: Asegura que todos los nodos del clúster tengan información actualizada sobre el estado del clúster.
- Comunicación distribuida: Permite que las acciones coordinadas (como la conmutación por error) se apliquen de forma consistente en todos los nodos.

## Arquitectura completa
```
       +---------------+                                 +---------------+               
       |    share      |                                 |    ActiveMQ   |
       +---------------+                                 +---------------+               
               |                                                 |
               |                                                 |
               +----------------------+--------------------------+
                                      |
                                      |
                                      V
                          +-----------------------+
                          |  Alfresco Community   |
                          |-----------------------|
                          | - Se conecta a        |
                          |   postgresql a través |
                          |   de HaProxy          |
                          +-----------------------+
                                      |
                                      |
                                      |
                                      V
                         +------------------------------+
                         |          HAProxy             |
                         |------------------------------|
                         | - Balanceador de carga       |
                         | - Verifica el estado de      |
                         |   pg-master y pg-slave       |
                         | - Puertos expuestos:         |
                         |   • 5432 (tráfico PostgreSQL)|
                         |   • 7000 (estadísticas)      |
                         +--------------+---------------+
                                        |
                                        | (Conexiones TCP entrantes)
                                        V
               +------------------------+------------------------+
               |                                                 |
       +---------------+                                  +---------------+
       |  pg-master    |                                  |   pg-slave    |
       | (PostgreSQL + |                                  | (PostgreSQL + |
       |   Patroni)    |                                  |   Patroni)    |
       | - API REST    |                                  | - API REST    |
       |   en 8008     |                                  |   en 8008     |
       | - Volumen de  |                                  | - Volumen de  |
       |   datos propio|                                  |   datos propio|
       +-------+-------+                                  +-------+-------+
               |                                                 |
               |                                                 |
               +----------------------+--------------------------+
                                      |
                                      | (Coordinación y estado)
                                      V
                              +-----------------------+
                              |  Zookeeper            |
                              |-----------------------|
                              | - Servicio de         |
                              |   coordinación        |
                              | - Elección de         |
                              |   líder (failover)    |
                              | - Comunicación con    |
                              |   Patroni en cada nodo|
                              +-----------------------+
```





## Flujo de conexión y coordinación:
1. Entrada de conexiones:
Alfresco Community se conecta a HAProxy en el puerto 5432.

2. Balanceo y verificación:
HAProxy verifica el estado de los nodos a través de sus APIs REST (puerto 8008) y dirige las conexiones hacia el nodo maestro (o en función de la configuración, también podría distribuir lecturas hacia los esclavos).

3. Coordinación del clúster: Patroni, instalado en cada nodo PostgreSQL, utiliza Zookeeper para:
    - Registrar el nodo en el clúster.
    - Realizar la elección de líder y gestionar el failover en caso de fallo del nodo maestro.
    - Actualizar y sincronizar la configuración entre todos los nodos.

4. Datos locales:
Cada nodo mantiene su propio volumen de almacenamiento, asegurando que cada instancia tenga su propia copia de la base de datos, mientras que la replicación se encarga de mantener la coherencia.

## ALfresco Community
La colleción de Alfresco detallada en esta colección es bastante simple y a modo de ejemplo, ya que se prescinde de los servicios de índices y también el de transformación.
Esta arquitectura, ha pasado un protocolo de pruebas de carga, miestras que varios cores de Alfresco están conectado conectado a los diferentes puntos de BBDD que tenemos:
    - HaProxy.
    - Pg-master.
    - Pg-slave.
Esta pruebas es para garantizar y ver de forma "visual" la replicación de los datos. Esto no es un clúster de Alfresco Community, solo está enfocado al entorno de BBDD.

