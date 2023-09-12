1. git clone 
2. to build image use $ docker-compose up -d --build   
3. open postman and run api localhost:3000/story get request 
4. also run some post request localhost:3000/story and send test:"mytext" in json payload then check in get request
 agin
 then stop docker-compose
 run $ docker-compose down and again run $ docker-compose up -d --build  and check get request api volume data still persist

Now lets start with kubernetes


Step 1: create deployment.yaml 
--------------------------------------------------------------
apiVersion: apps/v1
kind: Deployment
metadata:
  name: story-deployment
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: story
  template:
    metadata:
      labels:
        app: story
    spec:
      containers:
        - name: story
          image: appdockers/kube-data-demo  

--------------------------------------------------------------
service.yaml file
--------------------------------------------------------------
apiVersion: v1
kind: Service
metadata:
  name: story-service
spec:
  selector:
    app: story
  type: LoadBalancer  
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 3000    


Step 2: 
--------------------------------------------------------------

lets create repo on docker hub and create image and push on docker hub

$ docker build -t appdockers/kube-data-demo .   

$ docker push appdockers/kube-data-demo 

$ kubectl apply -f=service.yaml -f=deployment.yaml

$ kubectl get service 
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP   40h
story-service   ClusterIP   10.101.156.168   <none>        80/TCP    16s

then start service
$ minikube service story-service
o/p |-----------|---------------|-------------|---------------------------|
| NAMESPACE |     NAME      | TARGET PORT |            URL            |
|-----------|---------------|-------------|---------------------------|
| default   | story-service |          80 | http://192.168.49.2:31301 |
|-----------|---------------|-------------|---------------------------|
🏃  Starting tunnel for service story-service.
|-----------|---------------|-------------|------------------------|
| NAMESPACE |     NAME      | TARGET PORT |          URL           |
|-----------|---------------|-------------|------------------------|
| default   | story-service |             | http://127.0.0.1:61063 |
|-----------|---------------|-------------|------------------------|

Step 3: 
--------------------------------------------------------------

use the ip address http://127.0.0.1:61063 to call the api from postman 

both api will work the proble will come when the container cresh or restart 
all data will lost

to solve this we will use volume

kuber netes soppourt broad varity of types and driver for volume
we will cover 
emptyDir
hostPath
CSI


Step4 :
--------------------------------------------------------------

emptyDir Volume

will add error route in app.js to crash the container 
 add will add volume in deployment.yaml

emptyDir: 
          its create a empty dir when a pods will start and filled with data as long as the pod is live  
          container  will right data init and persist data if container will restart but when the pod is recreated its lost the data and initialize with empty directory
--------------------------------------------------------------
DEPLOYMENT.YAML  FILE


apiVersion: apps/v1
kind: Deployment
metadata:
  name: story-deployment
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: story
  template:
    metadata:
      labels:
        app: story
    spec:
      containers:
        - name: story
          image: appdockers/kube-data-demo  
      volumes:
        - name: story-volume
          emptyDir: {}          

Now only we added volume but its need to mount with container

--------------------------------------------------------------

apiVersion: apps/v1
kind: Deployment
metadata:
  name: story-deployment
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: story
  template:
    metadata:
      labels:
        app: story
    spec:
      containers:
        - name: story
          image: appdockers/kube-data-demo 
          volumeMounts:
            - mountPath: /app/story 
              name: story-volume 
      volumes:
        - name: story-volume
          emptyDir: {} 


but we can mount mltiple volume with container then we need to define name as well
see the example
--------------------------------------------------------------

apiVersion: apps/v1
kind: Deployment
metadata:
  name: story-deployment
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: story
  template:
    metadata:
      labels:
        app: story
    spec:
      containers:
        - name: story
          image: appdockers/kube-data-demo 
          volumeMounts:
            - mountPath: /app/story 
              name: story-volume
            - mountPath: /app/story 
              name: story-volume2  
      volumes:
        - name: story-volume
          emptyDir: {}    
        - name: story-volume2
          emptyDir: {}  
 
But what is you have multiple pods and if one pod will crash and second one will serve then
data will not in sync way 
second pod not have first pod data

hostPath
-----------------------------------------------------------------

to solve this we use hostPath
HostPath: hostPath will not always create a new empty path.
But it will simply share a path
and whatever is inside off that folder
with this path in the container.

When to use hostpath
if you have multiple replicas of pods then use hostpath which will share data using the path and it will not create empty dir

how to do configuration

apiVersion: apps/v1
kind: Deployment
metadata:
  name: story-deployment
spec:
  replicas: 2
  selector: 
    matchLabels:
      app: story
  template:
    metadata:
      labels:
        app: story
    spec:
      containers:
        - name: story
          image: appdockers/kube-data-demo:2 
          volumeMounts:
            - mountPath: /app/story 
              name: story-volume
      volumes:
        - name: story-volume
          hostPath:
            path: /data
            type: DirectoryOrCreate   


CSI 
-------------------------------------------------------------------------------------- 

Container storage storage it very flexible

CSI support with AWS EFS 
it can be used when you want to deploye your project on aws uning kubernetes and docker contianerization


Persistent Volumes

Volumes are destroyed when a Pod is removed
hostPath partially works around that in "One node" environment
Pod and Node independent Volumes are sometimes required
Persistent Volumes are pod or node independent


How the Persistent Volume work ?


The key idea however is,
that the volume will be detached from the Pod.
And that includes a total detachment
from the Pod life cycle.
Instead with persistent volumes,
we will have that Pod and Node independence
and as a cluster administrator
we will have full power over how this volume is configured.
We don't need to configure it multiple times
for different Pods and indifference deployment
YAML files or anything like that.
Instead, we'll be able to define it once
and then use it in multiple Pods if you want to.
So persistent volumes are built around the idea
of Pod and Node independence.

Why  we are using persistent 
First we create a perisitent volumen file then we create a claim file

host.pv.yaml file

apiVersion: v1
kind: PersistentVolume
metadata:
  name: host-pv
spec:
  capacity:
    storage: 4Gi        # 4Gi=4 gigabytes  this is capacity of pods to store data
  volumeMode: Filesystem     # it can be block as well but for this project we storeing file
  accessModes:
    - ReadWriteOnce
    # - ReadOnlyMany    # hostpath not support 
    # - ReadWriteMany   # hostpath not support 
  hostPath:
    path: /data
    type: DirectoryOrCreate



