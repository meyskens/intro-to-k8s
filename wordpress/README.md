# Wordpress on Kubernetes - a tutorial

Are you ready to sail the wild seas of the Kubernetes? Let's get started! 
This tutorial combines both the use of Helm charts as well as working with our own YAML manifests.

## Getting "a kube"
We need a local Kubernetes cluster to test on! I am personally a fan of [kind](https://kind.sigs.k8s.io/docs/user/quick-start/) as it works fast and is lightweight.
However on Windows systems without WSL [minikube](https://minikube.sigs.k8s.io/docs/start/) might also be a good option.

We also need [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and [Helm](https://helm.sh/docs/intro/quickstart/) installed!

### kind

We will be using Ingress here, this extra `--config` will enable the ingress being forwarded to your host

```bash
$ cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF

# Now let's install our ingress controller compoment
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/kind/deploy.yaml
kubectl get pod --all-namespaces # watch to see them all starting
```

### minikube

```bash
$ minikube start
$ minikube addons enable ingress
```

## Wordpress

We're going to use Wordpress as an example here, Wordpress needs a storage volume and a MySQL database.
Since managing databases is a solved issue we will be using a [Helm Chart](https://artifacthub.io/packages/helm/bitnami/mysql) to assist us here.
We will also need a volume to host our user uploads to our wordpress container.
```
 ----------         --------
(          )       |        | 
(          )       |        |
(  Volume  )  ---> |  MySql |
(          )       |        |
(          )       |        |
 ----------         --------

                        |
                        | ClusterIP Service  
                        |
                        v
----------         --------
(          )       |        |       @@@@@@@@@@@ 
(          )       |        |       @         @
(  Volume  )  ---> |   WP   |  ---> @ Ingress @ ---> User
(          )       |        |       @         @
(          )       |        |       @@@@@@@@@@@
 ----------         --------              
```

### MySQL

Let's start by our database! By looking at https://artifacthub.io/packages/helm/bitnami/mysql we learn a lot of options we can use to configure this.
We can write our own `values.yaml` file to set all these but using `-set` for now is shorter.

```bash
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm install wp-mysql bitnami/mysql --set auth.password=random --set auth.username=wp --set auth.database=wp
```

This will have installed everything our MySQL will need.

Let's look at what it has done!

```bash
kubectl get pod
NAME         READY   STATUS    RESTARTS   AGE
wp-mysql-0   1/1     Running   0          86s
```

We see we have one MySQL container running. But we also needed a service right?

```dns
$ kubectl get service
NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
wp-mysql              ClusterIP   10.96.17.79    <none>        3306/TCP   2m12s
wp-mysql-headless     ClusterIP   None           <none>        3306/TCP   2m12s
```
We see we have a `wp-mysql` service, which has an IP! The IP here can change and is only showed for information. Kubernetes also sets up DNS for us (yay!) so we can just call it using `wp-mysql` as hostname later.

But wait where does it store data?
```bash
$ kubectl get pvc
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-wp-mysql-0   Bound    pvc-19f8e65f-81be-4436-916c-2672f486199c   8Gi        RWO            standard       3m46s
```

PVC stands for Presistent Volume Claim and will get us a request for storage, minikube and kind will make some space on our local HDD. If you use Kubernetes in the cloud Kubernetes will actually order storage with your cloud provider for you!

At last we also have secrets! These are stored securely by Kubernetes and can be injected into other resources, here passwords are stored for example.
```bash
$ kubectl get secret
NAME                             TYPE                                  DATA   AGE
default-token-5g8dv              kubernetes.io/service-account-token   3      3s
sh.helm.release.v1.wp-mysql.v1   helm.sh/release.v1                    1      4s
wp-mysql                         Opaque                                2      4s
wp-mysql-token-74pxj             kubernetes.io/service-account-token   3      4s
```

We see that Helm amd Kubernetes themselves like to use secrets also! We are interested in the `wp-mysql` our Helm Chart created.

This command doesn't tell us much about `wp-mysql`, so let's use another one.
```bash
$ k describe secret wp-mysql
Name:         wp-mysql
Namespace:    default
Labels:       app.kubernetes.io/instance=wp-mysql
              app.kubernetes.io/managed-by=Helm
              app.kubernetes.io/name=mysql
              helm.sh/chart=mysql-8.2.3
Annotations:  <none>

Type:  Opaque

Data
====
mysql-password:       6 bytes
mysql-root-password:  10 bytes
```

`kubectl describe` is your best friend to read Kubernetes resources in human friendly format! 

```bash
$ kubectl describe secret wp-mysql
Name:         wp-mysql
Namespace:    default
Labels:       app.kubernetes.io/instance=wp-mysql
              app.kubernetes.io/managed-by=Helm
              app.kubernetes.io/name=mysql
              helm.sh/chart=mysql-8.2.3
Annotations:  <none>

Type:  Opaque

Data
====
mysql-password:       6 bytes
mysql-root-password:  10 bytes
```

We now know we have 2 data keys, these keys we can use later to get the data we want!

Our MySQL is now ready! Thanks to Helm we don't have to care too much about it, the experts who made the Helm Chart already put in years of expertise into making magic.

### Wordpress

MySQL was easy (no?)! There is also a Wordpress Helm Chart! But if you walk into a job where you just got hired to work on a Kubernetes cluster you probably won't find a Helm chart there. So let's do this by hand!

If you remember our over simplistic schematic from before we need storage.
Let's make a folder somewhere on your disk to make these YAML files, you always want to keep the YAML you write to be saved somewhere, so you can quickly restore on case of a cluster failure!

Copy the following code into `pvc.yaml` (or any other name you like more)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-persistent-storage # the name of our PVC
spec:
  accessModes:
    - ReadWriteOnce # This tells how many users can read/write or if it needs to be read only
  resources:
    requests:
      storage: 6Gi # how much storage we want
```

Now we need to apply it. You have 2 options:
```bash
kubectl create -f pvc.yaml
```
or
```bash
kubectl apply -f pvc.yaml
```

`kubectl create` will hard error if you already have the resources in your cluster, `kubectl apply` will try to update the resources if it finds them, the 2nd option is better if you want to fix anything you applied before. The first one however safeguards you against removing things if you named them the same by accident.

Let's see what we have created, shall we?

```bash
$ kubectl get pvc
NAME                    STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-wp-mysql-0         Bound     pvc-29cf2c86-2091-4f05-87ac-51dfa83c0dfb   8Gi        RWO            standard       30m
wp-persistent-storage   Pending                                                                        standard       11s
```

Pending? huh... `kubectl describe` to the rescue!

```bash
$ kubectl describe pvc wp-persistent-storage
Name:          wp-persistent-storage
Namespace:     default
StorageClass:  standard
Status:        Pending
Volume:        
Labels:        <none>
Annotations:   <none>
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      
Access Modes:  
VolumeMode:    Filesystem
Mounted By:    <none>
Events:
  Type    Reason                Age                From                         Message
  ----    ------                ----               ----                         -------
  Normal  WaitForFirstConsumer  15s (x7 over 92s)  persistentvolume-controller  waiting for first consumer to be created before binding
```

The `Events` section of the describe gives us recent updates on what is happening with our resources, Kubernetes here logs what it is doing with it.
`waiting for first consumer to be created before binding` is our error, Kubernetes is just waiting on us using the volume somewhere!

So let's use it! 
Our Deployment is our component that will create all our pods for us. This time we have quite a long one!
We now do a lot more, we tell Kubernetes to add secret data into our containers via [environment variables](https://en.wikipedia.org/wiki/Environment_variable) a common practice in containers.
It also will mount a volume to the container! Thanks to Kubernetes talking to your cloud provider in reality it can migrate the volume between physical servers for you :) 

Let's create `deployment.yaml`, we do a lot here so make sure to read the comments so you know what we are doing.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector: # the selector is for the deployment to know which pod's it owns, make sure to keep labels the same everywhere
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - image: wordpress:apache # we pull wordpress from hub.docker.com, `apache` is our tag meaning we pull that specific version, for WP this means the latest version with apache server built in
        imagePullPolicy: Always # this will alwatch check for a newer image when a pod stars, you can also set it to IfNotPresent so it only downloads it if not on disk already
        name: wordpress # name of the container, only used for you to know what is running
        env: # set environment variables 
        - name: WORDPRESS_DB_HOST # equivalent to `export WORDPRESS_DB_HOST=wp-mysql`
          value: wp-mysql # this is our service name from before! Kubernetes will automatically set up internal DNS to resolve service names to cluster IPs
        - name: WORDPRESS_DB_USER
          value: wp
        - name: WORDPRESS_DB_NAME
          value: wp
        - name: WORDPRESS_DB_PASSWORD
          valueFrom: # we can specify values by hand as above, or get them from secrets!
            secretKeyRef:
              name: wp-mysql
              key: mysql-password
        ports:  
        - containerPort: 80 # this gives the port 80 the name wordpress, it does not expose it to the outside world yet
          name: wordpress
        volumeMounts: #this part tells to mount the `wp-persistent-storage` to `/var/www/html`
        - name: wp-persistent-storage
          mountPath: "/var/www/html"
      volumes:
      - name: wp-persistent-storage # this is the actual definition of the `wp-persistent-storage` volume to tell it which PVC to use
        persistentVolumeClaim:
          claimName: wp-persistent-storage
```

Deploying this will take a few seconds as it will create the container and the volume. You can watch it setting up everything using `kubectl get pod` or `kubectl describe pod <pod name>` in more detail!
Tip: `kubectl describe pod -l app=wordpress` describes all pods with the label `app=wordpress`

Now we see it running and ready we have Wordpress running on Kubernetes and talking to our MySQL. Now we need to expose it to the "internet".

Let's create `service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80 # port on the service IP
      targetPort: wordpress # port on the container, can also be a number
  selector:
    app: wordpress
```

Now we have a service for our Wordpress install.
We are now going to expose it with our Ingress:
Let's create `ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: wordpress-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx" # tells us to use the nginx ingress controller
spec:
  rules:
    - host: wp.local # hostname to serve on, you can change this if you change it in your hosts file. If you use kind on linux you can make this `localhost` and skip the hosts file step below!
      http:
        paths:
          - path: /
            backend:
              serviceName: wordpress # tells which service to talk to
              servicePort: 80
```

Now let's look at it, you will see we got an `address` (might take a minute), 

```bash
$ kubectl get ingress
NAME                HOSTS       ADDRESS           PORTS   AGE
wordpress-ingress   localhost   192.168.122.158   80      19m
```

Take this IP and make a hosts entry on your system,
More info on how to do that:

* Windows: https://www.thewindowsclub.com/hosts-file-in-windows
* macOS: https://www.alphr.com/edit-hosts-file-mac-os-x/
* Linux: do i really have to tell you? okay here it is: https://www.makeuseof.com/tag/modify-manage-hosts-file-linux/

for example:
```
192.168.122.158 wp.local
```

If all goes right you can now open `wp.local` in your browser and enjoy wordpress!

## The End!

Congratulations you just make your first steps into the world of Kubernetes! There is a lot more to explore on these seas but this small introduction should help you to explore!