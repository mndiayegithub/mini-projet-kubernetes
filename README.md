----
Author : NDIAYE Mansour

Context : Bootcamp DevOps training, 12nd promotion

Training center : eazytraining.fr

Date : 24th March 2023

Linkedin : https://www.linkedin.com/in/mansour-ndiaye-64a04b161/

---

# Kubernetes mini - project
The aim of this project is to deploy wordpress and mysql in a kubernetes cluster following these steps : 
- Create a mysql deployment with only one replica,
- Create a service type clusterIP to expose mysql pods
- Create a wordpress deployment with all environmental variables to log into the mysql database
- Create a service type node port to expose the frontend wordpress

## Architecture

![image](https://user-images.githubusercontent.com/58290325/228014858-f2420df6-099b-4e57-8f93-85e8fecd8a89.png)


## Step 1 : Namespace creation

To separate my environments from others, I created a namespace `wordpress`:
```
apiVersion: v1
kind: Namespace
metadata: 
  name: wordpress
```

## Step 2 : Persistent Volumes and Persistent Volume Claims

According to my architecture, there's two persistent volumes (PV) and persistent volumes claim (pvc) for each service. The use of a persistent volume permit data persistence between changes and container restart.

The PV and PVC are configured to work as ReadWriteOnce. MySQL PV got a capacity of 4Gi whereas Wordpress PVC got a capacity of 3Gi.  

```
#MySQL PV and PVC
apiVersion: v1 
kind: PersistentVolume
metadata:
  name: mysql-pv
  namespace: wordpress
spec:
  storageClassName: manual
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 4Gi
  hostPath:
    path: /mnt/mysql-data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: wordpress
spec:
  resources:
    requests:
      storage: 3Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
```

```
#Wordpress PV and PVC
apiVersion: v1 
kind: PersistentVolume
metadata:
  name: wordpress-pv
  namespace: wordpress
spec:
  storageClassName: manual
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 2Gi
  hostPath:
    path: /mnt/mysql-wordpress
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  namespace: wordpress
spec:
  resources:
    requests:
      storage: 2Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
```

## Step 3 : Secrets

To avoid password exposition in data and improve the security, I used the object `Secret` to encrypt passwords with base64. 
For example to encrypt a string in base64, I do `echo -n 'password' | openssl base64` and it generates me the encrypted string in base64.

```
apiVersion: v1
kind: Secret
metadata:
  name: wordpress-secret
  namespace: wordpress
type: Opaque
data:
  WORDPRESS_PASSWORD: d29yZHByZXNz
  MYSQL_ROOT_PASSWORD: bXlzcWw=
```

## Step 4 : Deployments

Here's the deployments that I used for MySQL and WordPress. The secrets has been used to retrieve sensitive password and avoid data exposure. The recreate strategy type permits to ensure that old pods are always deleted before generating new ones.
```
#MySQL deployment
apiVersion: apps/v1
kind: Deployment 
metadata:
  name: wp-mysql
  namespace: wordpress
  labels:
    app: wordpress
    tier: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
      tier: backend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: backend
    spec:
      containers: 
      - name: mysql-container
        image: mysql
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_DATABASE
          value: wordpress
        - name: MYSQL_USER
          value: wordpress
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: wordpress-secret
              key: WORDPRESS_PASSWORD
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
             name: wordpress-secret
             key: MYSQL_ROOT_PASSWORD
        volumeMounts:
         - name: mysql-data
           mountPath: /var/lib/mysql
      volumes:
      - name: mysql-data
        persistentVolumeClaim:
          claimName: mysql-pvc
```

```
#Wordpress deployment
apiVersion: apps/v1
kind: Deployment 
metadata:
  name: wordpress
  namespace: wordpress
  labels:
    app: wordpress
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers: 
      - name: wordpress-container
        image: wordpress:latest
        ports:
        - containerPort: 80
        env:
        - name: WORDPRESS_DB_USER
          value: wordpress
        - name: WORDPRESS_DB_NAME
          value: wordpress
        - name: WORDPRESS_DB_HOST
          value: wp-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: wordpress-secret
              key: WORDPRESS_PASSWORD
        volumeMounts:
        - name: wordpress-data
          mountPath: /var/www/html
      volumes:
      - name: wordpress-data
        persistentVolumeClaim:
          claimName: wordpress-pvc
```

## Step 5 : Services

I created a clusterIP service (backend) to expose MySQL app inside the kubernetes cluster and a nodePort service (frontend) to access the app outside the cluster.

```
#MySQL clusterIP Service
apiVersion: v1 
kind: Service
metadata:
  name: wp-mysql
  namespace: wordpress
spec:
  selector:
    app: wordpress
    tier: backend
  type: ClusterIP
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
```

```
#Wordpress nodePort Service
apiVersion: v1 
kind: Service
metadata:
  name: wordpress-svc
  namespace: wordpress
spec:
  selector:
    app: wordpress
    tier: frontend
  type: NodePort
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 80
    nodePort: 30080
```

## Step 6 : Ingress

To access the app and delivery to a client, we can't let it accessible via **virtual IP : Port**. To delivery a good application to the client, I used `Ingress` object to expose the wordpress application with a custom domain name.

The following settings permits me to forward the app into the domain name http://dev-wordpress.fr.

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress-ingress
  namespace: wordpress
spec:
  rules:
  - host: dev.wordpress.fr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wordpress-svc
            port:
              number: 8080
```

## Step 7 : Deploy all in one

To automate the deployment of the application in any infrastructure, we can either copy paste all deployments in one YML file and launch it with :

`kubectl create -f deploy-wordpress-app.yml`

Or place all manifests in one folder and launch it with : 

`kubectl apply -f ./`

The two methods are useful because you still can delete all deployments in one command : 

`kubectl delete -f ./` or `kubectl delete -f deploy-wordpress-app.yml`

## Step 8 : Enjoying the application

After the app deployed in one command with the previous one, you can access Wordpress using the adress http://dev-wordpress.fr linked with a mysql database.

----

# Conclusion
