# my-fifth-project
![0](https://user-images.githubusercontent.com/64473684/85670821-e3427600-b6de-11ea-881a-3d700b5ae476.jpg)
**DevOps Assembly Lines: Continuous Monitoring**

**Objective is to create two containers for starting Prometheus and Grafana servers and Deploying them on the top of Kubernetes**



**Prometheus:**
![1](https://user-images.githubusercontent.com/64473684/85671253-521fcf00-b6df-11ea-988b-c208ffb90468.jpg)
1] Creating a Dockerfile for Prometheus container.
**Dockerfile:**
```javascript
FROM centos


RUN yum install wget -y

RUN wget https://github.com/prometheus/prometheus/releases/download/v2.19.0/prometheus-2.19.0.linux-amd64.tar.gz

RUN tar -xzf prometheus-2.19.0.linux-amd64.tar.gz

RUN mkdir /prom_data

CMD ./prometheus-2.19.0.linux-amd64/prometheus --config.file=/prometheus-2.19.0.linux-amd64/prometheus.yml --storage.tsdb.path=/prom_data && tail -f /dev/null
 
 ```
We create a new directory prom_data for storing metrics data generated by Prometheus. Build the above Docker image from the following command:

```javascript
docker build -t dockerninad07/prometheus:v1 .
```
Now create a deployment in Kubernetes for Prometheus containers. But before creating the Deployment, we need to create a Persistent Volume storage for storing the data generated by Prometheus, permanently. We create it as follows:

**2] PersistentVolume:**
```javascript
apiVersion: v1

kind: PersistentVolume

metadata:
  name: prom-pv
  labels:
    type: local

spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/sda1/data/prometheus"

```
Create the volume from the command:
```javascript
kubectl create -f prom_pv.yml
```
![0](https://user-images.githubusercontent.com/64473684/85672116-2ea95400-b6e0-11ea-88a8-e924324d5557.jpg)

**3] PersistentVolumeClaim:**
```javascript
apiVersion: v1

kind: PersistentVolumeClaim

metadata:
  name: prom-pv-claim

spec:
  storageClassName: manual
  accessModes: 
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi

```
Create the Claim:

```javascript
kubectl create -f prom_pv_claim.yml
```
﻿We can see that the Volume bounds with the claim
 
 ![0](https://user-images.githubusercontent.com/64473684/85672494-952e7200-b6e0-11ea-8595-e6b0982b6cb5.jpg)

Let us now create the required Deployment for Prometheus Container

**4] Deployment:**
```javascript
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prom-dep


spec:
  replicas: 3
  selector:
    matchLabels:
      app: prometheus


  template:
    metadata:
      name: prom-dep
      labels:
        app: prometheus


    spec:
      volumes:
        - name: prom-storage
          persistentVolumeClaim:
            claimName: prom-pv-claim
        
      containers:
        - name: prom-os
          image: dockerninad07/prometheus:v1
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - mountPath: "/prom_data"
              name: prom-storage
       
```
**Note: Make sure to push the created containers to your DockerHub repository so that the Node can pull the image at the time of Deployment.**
We mount the prom_data directory in our local system to the /mnt/sda1/data/prometheus directory in the Kubernetes Node to make it Persistent. Now create the deployment.

```javascript
kubectl create -f prom_dep.yml
```
After creating the deployment, we need to expose it to the outside world. Use the following command to expose:
```javascript
kubectl expose deployment prom-dep --port=8080 --type=NodePort
```
We now have our Deployment exposed. We need to do the same in case of Grafana container.

***Grafana**
![0](https://user-images.githubusercontent.com/64473684/85674602-9b255280-b6e2-11ea-9c31-fdc8835eed49.jpg)

***1] Dockerfile:***
```javascript
FROM centos


RUN yum install wget -y

RUN wget https://dl.grafana.com/oss/release/grafana-7.0.3-1.x86_64.rpm

RUN yum install grafana-7.0.3-1.x86_64.rpm -y


WORKDIR /usr/share/grafana


CMD /usr/sbin/grafana-server start && /usr/sbin/grafana-server enable && /bin/bash
```

Build the image and push it to the DockerHub repository:

```javascript
docker build -t dockerninad07/grafana:v1 .

docker push dockerninad07/grafana:v1
```
***2] PersistentVolume:***

```javascript

apiVersion: v1

kind: PersistentVolume

metadata:
  name: graf-pv
  labels:
    type: local

spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/sda1/data/grafana"
   ```

Create the volume:
```javascript

kubectl create -f graf_pv.yml
```
![0](https://user-images.githubusercontent.com/64473684/85675877-b80e5580-b6e3-11ea-9577-69374bd62cf5.jpg)

***3] PersistentVolumeClaim:***

```javascript

apiVersion: v1

kind: PersistentVolumeClaim

metadata:
  name: graf-pv-claim

spec:
  storageClassName: manual
  accessModes: 
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```
Create the volume:
```javascript
kubectl create -f graf_pv.yml
```
![0](https://user-images.githubusercontent.com/64473684/85677152-e80a2880-b6e4-11ea-9b8f-177ad64a8ffa.jpg)

We have mounted the /var/lib/grafana/ directory in the local system to the /mnt/sda1/data/grafana/ directory in the Kubernetes Node, thereby making it persistent.

Now create the Deployment.

***2] Deployment:***
```javascript
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graf-dep


spec:
  selector:
    matchLabels:
      app: grafana


  template:
    metadata:
      name: graf-dep
      labels:
        app: grafana
    
    spec:
      volumes:
        - name: graf-storage
          persistentVolumeClaim:
            claimName: graf-pv-claim

      containers:
        - name: graf-os
          image: dockerninad07/grafana:v1
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - mountPath: "/var/lib/grafana"
              name: graf-storage

      restartPolicy: Always 
```
***Note: Do not create Replicas for Grafana. It throws an Authority error while trying to login.***

Start the deployment and expose it:
```javascript
kubectl create -f graf_dep.yml

kubectl expose deployment graf-dep --port=3000 --type=NodePort
```
We can see that the containers have been deployed:
      
![0](https://user-images.githubusercontent.com/64473684/85678138-d1b09c80-b6e5-11ea-86ae-27a147ee795b.jpg)
As we now have our Deployments, we are ready to integrate both the tools



**Monitoring:**
Use the IP address of the Kubernetes Node along with the Port number generated for the Prometheus Deployment






    


  










