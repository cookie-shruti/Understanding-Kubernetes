
A complete walkthrough of building, deploying, and persisting a Node.js + MongoDB app on Kubernetes using Docker, `kubectl`, Persistent Volumes, and Minikube.

---

## 📦 Step 1: Build and Push Docker Image


docker build -t node-db-app:v3 .
docker tag node-db-app:v3 docker.io/<your-dockerhub>/node-db-app:v3
docker push docker.io/<your-dockerhub>/node-db-app:v3

## 🧾 Step 2: Check for Previous Services
kubectl get all

## 📁 Step 3: MongoDB Deployment

mongo-db.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-app
  template:
    metadata:
      labels:
        app: mongo-app
    spec:
      containers:
      - name: mongo-app
        image: mongo:latest


Apply the file:

kubectl apply -f mongo-db.yml


## 🧾 Step 4: Deploy Node.js App
node-app.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: noder-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: noder-app
  template:
    metadata:
      labels:
        app: noder-app
    spec:
      containers:
      - name: noder-app
        image: philippaul/node-mongo-db:04
        
Apply the file and expose the service:

kubectl apply -f node-app.yml

minikube service service-node-app

## 🔁 Step 5: Change Replicas to 2

Modify replicas: 2 in node-app.yml, then re-apply:

kubectl apply -f node-app.yml
✅ This runs multiple replicas of the same container.

## 📁 Step 6: Volume Setup
host-pv.yml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: host-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  hostPath:
    path: /data/
    type: DirectoryOrCreate

Apply and check:

kubectl apply -f host-pv.yml
kubectl get pv

host-pvc.yml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: host-pvc 
spec:
  volumeName: host-pv
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

Apply and check:


kubectl apply -f host-pvc.yml
kubectl get pvc


## 🔁 Step 7: Attach Volume to MongoDB

Update mongo-db.yml with volume configuration:


containers:
  - name: mongo-app
    image: mongo:latest
    volumeMounts:
      - mountPath: /data/db
        name: mongo-vol
volumes:
  - name: mongo-vol
    persistentVolumeClaim:
      claimName: host-pvc

Reapply:

kubectl apply -f mongo-db.yml
🔁 Step 8: Volume Testing
Delete the MongoDB deployment:

kubectl delete deployment mongo-app
Reapply:

kubectl apply -f mongo-db.yml
✅ If data remains, volumes are working fine.

## 🤝 Step 9: MongoDB Service

apiVersion: v1
kind: Service
metadata:
  name: service-mongodb
spec:
  selector:
    app: mongo-app
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017

## 🔗 Step 10: Multi-Container App (Node + Mongo + ENV)
node-app.yml updated:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: noder-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: noder-app
  template:
    metadata:
      labels:
        app: noder-app
    spec:
      containers:
      - name: noder-app
        image: philippaul/node-mongo-db:04
        env:
        - name: MONGO_HOST
          valueFrom:
            configMapKeyRef:
              name: mongo-config
              key: MONGO_HOST
        - name: MONGO_PORT
          valueFrom:
            configMapKeyRef:
              name: mongo-config
              key: MONGO_PORT

service-noder-app.yml

apiVersion: v1
kind: Service
metadata:
  name: service-noder-app
spec:
  selector:
    app: noder-app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 3000
  type: LoadBalancer


## 🧠 Summary of Concepts Used

## Concept	Description

Deployment	Runs your containers in pods
Services	Exposes apps within cluster or outside
PersistentVolume (PV)	Reserves storage on host
PersistentVolumeClaim	Requests specific storage
VolumeMount	Mounts storage into a container path
ReplicaSet	Ensures desired number of pod replicas
Environment Variables	Connect to external resources like MongoDB



