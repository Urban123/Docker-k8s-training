In this lab, we will declare and create a cohesive multi-application stack with services through Kubernetes.

Chapter Details

| Chapter Goal | Declare, deploy, and verify a cohesive application deployment |
|---|---|
|   | 1. Application Deployment Architecture  |
|   | 2. Kubernetes Secrets |
|   | 3. Create a Persistent Volume  |
| Chapter Sections  | 4. Create the MySQL Deployment  |
|   | 5. Create a Service for MySQL |
|   | 6. Create the Wordpress Deployment  |
|   | 7. Create a Service for Wordpress  |


### Application Deployment Architecture
In this lab we will deploy the Wordpress content management system with a MySQL backend. Both applications will reside within separate Pods. We will declare and create a Service for MySQL which Wordpress will use to connect to the database. We will also use a Kubernetes secret to hold our database connection information such as username, passwords, and database name.

Since MySQL is persisting the data on behalf of Wordpress, we will provision a persistent volume that will be used to house the data for MySQL.

To be able to access the Wordpress CMS from outside the Kubernetes cluster, we will also create a Service for Wordpress to allow for easy access. Alright, let’s get to it.

### Kubernetes Secrets
Kubernetes secrets allow users to define sensitive information outside of containers and expose that information to containers through environment variables as well as files within Pods. In this section we will declare and create secrets to hold our database connection information that will be used by Wordpress to connect to its backend database.


**Step 1** Open up two terminal windows. We will use one window to generate encoded strings that will contain our sensitive data. The other window will be used to create the secrets YAML declaration.

**Step 2** In the first terminal window, execute the following commands to encode our strings:

```
$ echo -n "wordpress_db" | base64
d29yZHByZXNzX2Ri

$ echo -n "wordpress_user" | base64
d29yZHByZXNzX3VzZXI=

$ echo -n "wordpress_pass" | base64
d29yZHByZXNzX3Bhc3M=

$ echo -n "rootpass123" | base64
cm9vdHBhc3MxMjM=
```

**Step 3** In the second terminal window, create a directory where we will store our declarations and create a file named wordpress-db-secrets.yaml with the contents below, making sure to copy and paste the encoded values from the previous step into the appropriate fields:
```
$ mkdir -p ~/k8s-artifacts/wordpress
$ cd ~/k8s-artifacts/wordpress
$ vim wordpress-db-secrets.yaml

apiVersion: v1
kind: Secret
metadata:
  name: wordpress-db-secrets
type: Opaque
data:
  dbname: d29yZHByZXNzX2Ri
  dbuser: d29yZHByZXNzX3VzZXI=
  dbpassword: d29yZHByZXNzX3Bhc3M=
  mysqlrootpassword: cm9vdHBhc3MxMjM=
```
**Step 4** Create the secret:
```
$ kubectl create -f wordpress-db-secrets.yaml
secret "wordpress-db-secrets" created
```

**Step 5** Verify creation and get details:
```
$ kubectl get secrets
NAME                   TYPE                                  DATA      AGE
default-token-ahhjz    kubernetes.io/service-account-token   3         37d
wordpress-db-secrets   Opaque                                4         3h

$ kubectl describe secrets/wordpress-db-secrets
Name:           wordpress-db-secrets
Namespace:      default
Labels:         <none>
Annotations:    <none>

Type:   Opaque

Data
====
dbname:             12 bytes
dbpassword:         14 bytes
dbuser:             14 bytes
mysqlrootpassword:  11 bytes
```

### Create a Persistent Volume
We will use a persistent volume to provide the underlying storage target for our MySQL database. In our environment, we will use a HostPath device, which will just pass a directory from the host into a container for consumption.

**Notes**
If you are using a hosted Kubernetes environment, such as on Google Cloud Platform, persistent volumes can be created as GCE Persistent disks rather than HostPath. One downfall to HostPath is that it only works with single node Kubernetes clusters, as there is no guarantee Pods will get created where storage exists when multiple node deployments are used.


**Step 1** Create a directory we will use as our HostPath persistent disk:
```
$ sudo mkdir -p /data/mysql-wordpress
```
**Step 2** Create a file under ~/k8s-artifacts/wordpress named mysql-persistentvol.yaml with the contents below:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
  labels:
    vol: mysql
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /data/mysql-wordpress
```

**Step 3** Create a Kubernetes persistent volume and verify its status is set to Available:
           
```
$ kubectl create -f mysql-persistentvol.yaml
persistentvolume "mysql-pv" created 

$ kubectl describe pv/mysql-pv
Name:           mysql-pv
Labels:         vol=mysql
Status:         Available
Claim:
Reclaim Policy: Retain
Access Modes:   RWO
Capacity:       5Gi
Message:
Source:
  Type:         HostPath (bare host directory volume)
  Path:         /data/mysql-wordpress
No events.
```

### Create the MySQL Deployment
We are now ready to declare and have Kubernetes create our deployment for MySQL.

**Step 1** Create a file named mysql-deployment.yaml in the ~/k8s-artifacts/wordpress directory with the contents below:
```
kind: PersistentVolumeClaim

apiVersion: v1

metadata:

  name: mysql-pv-claim

spec:

  accessModes:

    - ReadWriteOnce

  resources:

    requests:

      storage: 5Gi

  selector:

    matchLabels:

      vol: "mysql"

---

apiVersion: extensions/v1beta1

kind: Deployment

metadata:

  name: mysql-deployment

spec:

  replicas: 1

  strategy:

    type: RollingUpdate

  template:

    metadata:

      labels:

        app: mysql

        track: production

    spec:

      containers:

      - name: "mysql"

        image: "mysql:5.6"

        ports:

        - containerPort: 3306

        volumeMounts:

        - mountPath: "/var/lib/mysql"

          name: mysql-pd

        env:

        - name: MYSQL_ROOT_PASSWORD

          valueFrom:

            secretKeyRef:

              name: wordpress-db-secrets

              key: mysqlrootpassword

        - name: MYSQL_USER

          valueFrom:

            secretKeyRef:

              name: wordpress-db-secrets

              key: dbuser

        - name: MYSQL_PASSWORD

          valueFrom:
            secretKeyRef:
              name: wordpress-db-secrets
              key: dbpassword
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: wordpress-db-secrets
              key: dbname
      volumes:
        - name: mysql-pd
          persistentVolumeClaim:
            claimName: mysql-pv-claim
```
Alright, so there is quite a bit of stuff happening in the above YAML. Let us try to break it down piece by piece.

We are declaring two resources within this single YAML document. The first resource is a persistent volume claim to our previously created persistent volume. We are making the association to the persistent volume through our label selector.

The second resource we are declaring is our deployment for MySQL. We are specifying our standard information such as container name, image to use, and exposed port to access. We are then mounting a volume to the/var/lib/mysql directory in the container, where the volume to mount is named mysql-pd and is declared at the bottom of this document.

We are also declaring environment variables to initialize. The MySQL image we are using that is available on Docker Hub supports environment variable injection. The four environment variables we are initializing are defined and used within the Docker image itself. The values we are setting these environment variables to are all referencing different keys we set in our Secret earlier on. When this container starts up, we will automatically have MySQL configured with the desired root user password, and we will also have the database for Wordpress created with appropriate access granted for our Wordpress user.


**Step 2** Create the resources and verify everything was created successfully:
```
$ kubectl create -f mysql-deployment.yaml
persistentvolumeclaim "mysql-pv-claim" created
deployment "mysql-deployment" created

$ kubectl get pv
NAME      CAPACITY  ACCESSMODES   STATUS  CLAIM                   REASON    AGE
mysql-pv  5Gi       RWO           Bound   default/mysql-pv-claim            14m

$ kubectl get pvc
NAME            STATUS  VOLUME    CAPACITY  ACCESSMODES AGE
mysql-pv-claim  Bound   mysql-pv  0                     2m

$ kubectl get deployments
NAME              DESIRED   CURRENT   UP-TO-DATE  AVAILABLE   AGE
mysql-deployment  1         1         1           1           2m
```

**Step 3** Let’s verify the device was properly mapped into the container and the database was successfully created:
```
$ kubectl get pods
NAME                    READY   STATUS    RESTARTS  AGE
mysql-deployment-<id>   1/1     RUNNING   0         2m

$ kubectl exec -it mysql-deployment-<id> bash

root@mysql-deployment:/# mount | grep /var/lib/mysql
tmpfs on /var/lib/mysql type tmpfs (rw,relatime,size=917852k)

root@mysql-deployment:/# mysql -u root -p
Enter password: rootpass123

mysql> show databases;
+-----------------+
| Database        |
+-----------------+
...
| wordpress_db    |
+-----------------+

mysql> show grants for wordpress_user;
+---------------------------------------------------------------------------+
| Grants for wordpress_user@%                                               |
+---------------------------------------------------------------------------|
| GRANT USAGE ON *.* TO 'wordpress_user'@'%' IDENTIFIED BY PASSWORD <pass>  |
| GRANT ALL PRIVILEGES ON `wordpress_db`.* TO 'wordpress_user'@'%'          |
+---------------------------------------------------------------------------+
```

### Create a Service for MySQL
As we know, Pods are ephemeral. They come and go and needed, with each newly created Pod receiving a new and different IP address. Because of this, for our Pods to reliably communicate with one another, we cannot staticly configure them to talk directly to a Pod over its assigned IP address. We need an IP address that is decoupled from that of a Pod and that never changes, and this is exactly what Kubernetes services offer.

In this section we will go ahead and declare and create a service for MySQL. Let’s get to it!

**Step 1** Create a file named mysql-service.yaml in the ~stack/k8s-artifacts/wordpress directory and populate it with the contents below:
```
apiVersion: v1
kind: Service
metadata:
  name: mysql-internal
spec:
  ports:
    - port: 3306
      protocol: TCP
      targetPort: 3306
  selector:
    app: mysql
```

**Step 2** Create the service and verify its properties:
```
$ kubectl create -f mysql-service.yaml
service "mysql-internal" created

$ kubectl describe svc/mysql-internal
Name:               mysql-internal
Namespace:          default
Labels:             <none>
Selector:           app=mysql
Type:               ClusterIP
IP:                 ...
Port:               <unset> 3306/TCP
Endpoints:          172.17.0.3:3306
Session Affinity:   None
No events.

$ kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS  AGE   IP
mysql-deployment-<id>   1/1     RUNNING   0         30m   172.17.0.3
```
From the output above, we can verify the service was correctly mapped to to the pod for our MySQL deployment in that the Endpoints IP for the service aligns with the IP for the MySQL Pod.

### Create the Wordpress Deployment
Now that MySQL is deployed, persisting data to a volume, and is accessible over a Kubernetes service, we are ready to deploy our application - Wordpress. Wordpress is a very popular content management system written in PHP that has several different image versions available on Docker hub. We will use one of the pre-existing images that bundles Wordpress together with Apache, PHP, and PHP modules in our exercise.

**Step 1** Let’s first declare our Wordpress deployment. In the ~/k8s-artifacts/wordpress directory populate a file named wordpress-deployment.yaml with the following contents:
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: wordpress-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: wordpress
        track: production
    spec:
      containers:
      - name: "wordpress"
        image: "wordpress:4.5-apache"
        ports:
        - containerPort: 80
        env:
        - name: WORDPRESS_DB_HOST
          value: "mysql-internal"
        - name: WORDPRESS_DB_USER
          valueFrom:
            secretKeyRef:
              name: wordpress-db-secrets
              key: dbuser
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: wordpress-db-secrets
              key: dbpassword
        - name: WORDPRESS_DB_NAME
          valueFrom:
            secretKeyRef:
              name: wordpress-db-secrets
              key: dbname
```
One of the key things we are doing in the YAML above is initializing the environment variable WORDPRESS_DB_HOST to a value of mysql-internal. This is how we are telling the Wordpress application to access its database through the Kubernetes service we created in the previous section.

When we created that service, Kubernetes created a DNS record mapping the IP of the service to a name equal to the name of the service itself, which as you recall was mysql-internal.

**Step 2** Create and verify the deployment:
           
```
$ kubectl create -f wordpress-deployment.yaml
deployment "wordpress-deployment" created

$ kubectl get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE  AVAILABLE   AGE
mysql-deployment      1         1         1           1           45m
wordpress-deployment  3         3         3           3           5m
```

**Step 3** Get a list of created Pods:
```
$ kubectl get pods
NAME                      READY   STATUS    RESTARTS  AGE
mysql-deployment-425      1/1     Running   0         45m
wordpress-deployment-hq4  1/1     Running   0         6m
wordpress-deployment-jnr  1/1     Running   0         6m
wordpress-deployment-l13  1/1     Running   0         6m
```
Make note of the name of one of the Wordpress Pods from the output above.

**Step 4** Execute a shell within one of the Wordpress Pods found in the previous step:
```
$ kubectl exec -it <pod name> bash

root@wordpress# getent hosts mysql-internal
10.0.0.248      mysql-internal.default.svc.cluster.local
# Your IP may be different
```
The above output verifies that mysql-internal can be resolved through DNS to the ClusterIP address that was assigned to the MySQL service. Good stuff!

Now let’s verify Wordpress was properly configured:

```
root@wordpress# grep -i db /var/www/html/wp-config.php
define('DB_NAME', 'wordpress_db');
define('DB_USER', 'wordpress_user');
define('DB_PASSWORD', 'wordpress_pass');
define('DB_HOST', 'mysql-internal');
...
```
That all looks good as well. Nice work champ!
### Create a Service for Wordpress
The final thing required is to expose the Wordpress application to external users. For this, we again will need a service. In this example, we will expose a high numbered port on the Node running our application, and DNAT it to port 80 of our container. It will allow us to access the application, but it probably is not the approach one would take in production, especially if Kubernetes is hosted by a service provider.

Kubernetes can integrate with Load Balancing services offered by platforms such as GCE and AWS. If you are using either of those, then that would be one approach to take to take advantage of the load balancing functionality offered in those platforms.

**Step 1** Create a file named wordpress-service.yaml in the ~/k8s-artifacts/wordpress directory. Populate the file with the contents below:
```
apiVersion: v1
kind: Service
metadata:
  name: wordpress-service
  labels:
    app: wordpress
    track: production
spec:
  type: NodePort
  ports:
    - port: 80
      nodePort: 30080
  selector:
    app: wordpress
    track: production
``` 

**Step 2**  Create the service and verify its status:
```
$ kubectl create -f wordpress-service.yaml
service "wordpress-service" created

$ kubectl describe svc/wordpress-service
Name:               wordpress-service
Namespace:          default
Labels:             app=wordpress
                    track=production
Selector:           app=wordpress,track=production
Type:               NodePort
IP:                 ...
Port:               <unset> 80/TCP
NodePort:           <unset> 30080/TCP
Endpoints:          ...:80,...:80,...:80
Session Affinity:   None
No events.
```

 Open up your browser and navigate to http://<lab IP>:30080. You can follow the installation wizard to get Wordpress up and running through the browser.

Congratulations on deploying a cohesive multi-pod application stack with Kubernetes!