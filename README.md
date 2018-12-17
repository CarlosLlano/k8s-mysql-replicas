# Réplicas de Mysql en Kubernetes

## Introducción

Cuando se crean aplicaciones basadas en contenedores, un aspecto muy importante a tener en cuenta es lograr la persistencia de los datos. 

En el caso de un contenedor mysql, la solución es crear un volumen externo que almacene la información de la carpeta de datos de mysql 
(/var/lib/mysql). De esta forma, independiente de lo que pase con el contenedor, los datos siempre serán persistentes.

Esta solución es adecuada cuando se tiene solo una réplica de un contenedor mysql. Pero ¿qué pasa cuando se quieren tener más réplicas? 
Por ejemplo, para una aplicación de alta demanda y con gran número de transacciones, donde interesa hacer balanceo de carga para la base 
de datos.

Mapear todas las carpetas de datos de las réplicas de msyql al mismo volumen no es posible ya que se obtienen errores como los siguientes:

![captura de pantalla 2018-12-16 a la s 11 10 20 p m](https://user-images.githubusercontent.com/17281733/50066098-df4ba480-0187-11e9-8284-ea2b0cdb2384.png)


De estos errores se desprende que cada réplica de mysql debe tener su propia carpeta de datos para funcionar correctamente. 
¿Cómo lograr que cada réplica tenga los mismos datos y que todo cambio en una réplica se replique en todas las demás? 

Para ello, existe el software **percona xtradb cluster** (https://www.percona.com/software/mysql-database/percona-xtradb-cluster) 
que es una solución para crear clusters de mysql, que garantiza la consistencia de los datos entre todas las réplicas del cluster. 
Esta solución tiene una imagen pública para docker (https://hub.docker.com/r/percona/percona-xtradb-cluster).

A continuación se mostrarán los pasos para crear un cluster de mysql en kubernetes.

## Arquitectura General

![captura de pantalla 2018-12-16 a la s 10 17 34 p m](https://user-images.githubusercontent.com/17281733/50064661-67c64700-0180-11e9-85b3-adfe728a9b2f.png)

Para persistir los datos se usará un servidor NFS de IBM Cloud (File Storage - https://www.ibm.com/cloud/file-storage), el cual debe crearse
previamente.


## 1. Creación de los volumenes

![pv](https://user-images.githubusercontent.com/17281733/50065379-3fd8e280-0184-11e9-8e11-b9691294a43e.png)

**Persistent Volume**

Reemplazar los valores de **server** y **path** con lo datos del File Storage propio:

```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-mysql-0
  labels:
    app: mysql
    podindex: "0"
  annotations:
    volume.beta.kubernetes.io/storage-class: ""
spec:
  capacity:
   storage: "20Gi" # volume capacity
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: "fsf-dal1301e-fz.adn.networklayer.com" # hostname or IP address of the NFS server
    path: "/SL02SV1344071_1/data01/small/mysql-0" # mysql-replica-0 data directory path
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-mysql-1
  labels:
    app: mysql
    podindex: "1"
  annotations:
    volume.beta.kubernetes.io/storage-class: ""
spec:
  capacity:
   storage: "20Gi" # volume capacity
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: "fsf-dal1301e-fz.adn.networklayer.com" # hostname or IP address of the NFS server
    path: "/SL02SV1344071_1/data01/small/mysql-1" # mysql-replica-1 data directory path
```

**Persistent Volume Claim**

```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-mysql-0
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi # volume capacity
  selector:
    matchLabels:
      app: mysql
      podindex: "0"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-mysql-1
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi # volume capacity
  selector:
    matchLabels:
      app: mysql
      podindex: "1"
 ```

En la carpeta kubernetes, ejecutar los siguientes comandos:
```
kubectl apply -f 1.pvs.yml
kubectl apply -f 2.pvcs.yml
```

## 2. Creación del cluster de mysql

**Creación de la imagen**

En la carpeta docker-pxc, ejecutar los siguientes comandos reemplazando *dockerhub-username* por el usuario de dockerhub propio:
```
docker build -t <dockerhub-username>/percona-xtradb-cluster:5.7.19 .
docker push <dockerhub-username>/percona-xtradb-cluster:5.7.19
```

**Cluster Mysql y Servicio de Balanceo**

![mysql](https://user-images.githubusercontent.com/17281733/50065382-423b3c80-0184-11e9-96d8-9cf63caf8957.png)

```yml
apiVersion: apps/v1beta2
kind: StatefulSet
metadata:
  name: mysql
spec:
  replicas: 2
  selector:
    matchLabels:
      # Label selector that determines which Pods belong to the StatefulSet. 
      # Must match spec -> template -> metadata -> labels
      app: percona-galera 
  #service name that routes traffic to the pods created by the statefulsets
  serviceName: mysql-db 
  template:
    metadata:
      labels:
        app: percona-galera # Pod template's label selector
    spec:
      containers:
      - name: percona-galera
        imagePullPolicy: Always
        image: <dockerhub-username>/percona-xtradb-cluster:5.7.19>
        env:
        - name: CLUSTER_NAME
          value: percona-galera
        - name: MYSQL_ROOT_PASSWORD
          value: not-a-secure-password
        - name: K8S_SERVICE_NAME
          value: percona-galera-xtradb
        - name: LOG_TO_STDOUT
          value: "true"
        - name: DEBUG
          value: "true"
        livenessProbe: 
              # Indicates whether the Container is running. 
              # If the liveness probe fails, the kubelet kills the Container, and the Container is subjected to its restart policy.
              # If a Container does not provide a liveness probe, the default state is Success.
          exec:
            command: ["mysqladmin", "-p$(MYSQL_ROOT_PASSWORD)", "ping"]
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
              # Indicates whether the Container is ready to service requests. 
              # If the readiness probe fails, 
              # the endpoints controller removes the Pod’s IP address from the endpoints of all Services that match the Pod. 
              # The default state of readiness before the initial delay is Failure. 
              # If a Container does not provide a readiness probe, the default state is Success.
          exec:
            command: ["mysql", "-h", "127.0.0.1", "-p$(MYSQL_ROOT_PASSWORD)", "-e", "SELECT 1"]
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
        volumeMounts:
        - name: pvc
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: pvc
    spec:
      accessModes: [ "ReadWriteMany" ]
      resources:
        requests:
          storage: 20Gi
---
# HEADLESS SERVICE
apiVersion: v1
kind: Service
metadata:
  name: percona-galera-xtradb
spec:
  clusterIP: None
  ports:
  - name: galera-replication
    port: 4567
  - name: state-transfer
    port: 4568
  - name: state-snapshot
    port: 4444
  selector:
    app: percona-galera
---
# SERVICIO CON BALANCEO
apiVersion: v1
kind: Service
metadata:
  name: mysql-db
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: percona-galera
```

En la carpeta kubernetes, en el script 3.mysql, reemplazar el nombre de la imagen *dockerhub-username/percona-xtradb-cluster:5.7.19*
con el dado en el paso anterior y ejecutar el siguiente comando:
```
kubectl apply -f 3.mysql.yml
```

Para probar el balanceo:
```
kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
   bash -ic "while sleep 1; do mysql -h mysql-db -uroot -pnot-a-secure-password -e 'select @@hostname'; done"
```


## Recursos
 - **Mysql clustering solution - Percona xtradb cluster** (https://www.percona.com/software/mysql-database/percona-xtradb-cluster)
 - **Percona xtradb cluster para docker** (https://hub.docker.com/r/percona/percona-xtradb-cluster)
 - **NFS File Storage** (https://www.ibm.com/cloud/file-storage)
