In this lab, we will install a single-node Kubernetes cluster and explore it basic features.

Chapter Details

| Chapter Goal | Install and use a single-node Kubernetes cluster |
|---|---|
|   | 1. Install a single-node Kubernetes cluster  |
|   | 2. Kubernetes client |
|   | 3. Create a Pod  |
| Chapter Sections  | 4. Create a Replication Controller  |
|   | 5. Create a Service |
|   | 6. Delete a Service, Controller, Pod  |
|   | 7. Create a Deployment  |
|   | 8. Create a Job  |


### Install a single-node Kubernetes cluster
Minikube (https://github.com/kubernetes/minikube) is a tool that allows running a single-node Kubernetes cluster in a virtual machine. It can be used on GNU/Linux os OS X and requires VirtualBox, KVM (for Linux), xhyve (OS X), or VMware Fusion (OS X) to be installed on your computer. Minikube creates a new virtual machine with GNU/Linux, installs and configures Docker and Kubernetes, runs a Kubernetes cluster. You can use Minikube on your laptop to explore Kubernetes features.

**Step 1** Install minikube
``` 
$ curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo cp minikube /usr/local/bin/ && rm minikube
```

**Step 2** Start a local Kubernetes cluster:

```
$ minikube start --vm-driver=none
Done. It may take about a minute before apiserver is up.
```

The local Kubernetes cluster is up and running. Check that kubectl can connect to the Kubernetes cluster:
```
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"4", ...
Server Version: version.Info{Major:"1", Minor:"4", ...
```

###  Kubernetes client
The kubectl tools allows connecting to multiple Kubernetes clusters. A set of parameters, for example, the address the Kubernetes API server, credentials is called a context. You can define several contexts and specify which context to use to connect to a specific cluster. Also you can specify a default context to use.

**Step 1** Use kubectl config view to view the current kubectl configuration:
```
$ kubectl config view
apiVersion: v1
clusters: []
contexts: []
current-context: ""
kind: Config
preferences: {}
users: []
```

> Don't do next exercises! Examples below are just to show, how to connect to some other cluster.


**Step 2**  Specify the address of the Kubernetes API server via kubectl command line:
```
$ kubectl -s http://localhost:8080 version
Client Version: version.Info{Major:"1", Minor:"4", ...
Server Version: version.Info{Major:"1", Minor:"4", ...
```
**Step 3** Let’s define a new context, set the address of the Kubernetes API server, and make this context the default one:

```
$ kubectl config set-cluster default --server=http://localhost:8080
cluster "default" set.

$ kubectl config set-context default --cluster=default
context "default" set.

$ kubectl config use-context default
switched to context "default".
```
**Step 4** Use kubectl config view to view the current kubectl configuration:
```
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    server: http://localhost:8080
  name: default
contexts:
- context:
    cluster: default
    user: ""
  name: default
current-context: default
kind: Config
preferences: {}
users: []
```

### Create a Pod
We are going to begin declaring Kubernetes resources by writing YAML files containing resource definitions. To make our lives easier, let’s create a simple .vimrc configuration file that will deal with indentation and whitespaces the way we want.

**Step 1** Create vim configuration file at /home/<user>/.vimrc and populate it with the contents below:
```
set expandtab
set tabstop=2
```

**Step 2** Define a new pod in the file echoserver-pod.yaml:
```
apiVersion: v1
kind: Pod
metadata:
  name: echoserver
spec:
  containers:
  - name: echoserver
    image: gcr.io/google_containers/echoserver:1.4
    ports:
    - containerPort: 8080
```
We use the existing image echoserver. This is a simple server that responds with the http headers it received. It runs on nginx server and implemented using lua in the nginx configuration: https://github.com/kubernetes/contrib/tree/master/ingress/echoheaders

**Step 3** Create the echoserver pod:
```
$ kubectl create -f echoserver-pod.yaml
pod "echoserver" created
```

**Step 4** Use kubectl get pods to watch the pod get created:
```
$ kubectl get pods
NAME         READY     STATUS    RESTARTS   AGE
echoserver   1/1       Running   0          5s
```

**Step 5** Now let’s get the pod definition back from Kubernetes:
```
$ kubectl get pods echoserver -o yaml > echoserver-pod-created.yaml
```
Compare echoserver-pod.yaml and echoserver-pod-created.yaml to see additional properties that have been added to the original pod definition.

### Create a Replication Controller
**Step 1** Define a new replication controller for 2 replicas for the pod echoserver. Create a new file echoserver-rc.yaml with the following content:
```
apiVersion: v1
kind: ReplicationController
metadata:
  name: echoserver
spec:
  replicas: 2
  selector:
    app: echoserver
  template:
    metadata:
      name: echoserver
      labels:
        app: echoserver
    spec:
      containers:
      - name: echoserver
        image: gcr.io/google_containers/echoserver:1.4
        ports:
        - containerPort: 8080
```

**Step 2** Create a new replication controller:
           
```
$ kubectl create -f echoserver-rc.yaml
replicationcontroller "echoserver" created
```

**Step 3** Use kubectl get replicationcontrollers to list replication controllers:
```
$ kubectl get replicationcontrollers
NAME         DESIRED   CURRENT   AGE
echoserver   2         2         1m
```
**Step 4** Use kubectl get pods to list pods:
```
$ kubectl get pods
NAME               READY     STATUS    RESTARTS   AGE
echoserver         1/1       Running   0          37m
echoserver-obzuw   1/1       Running   0          46s
echoserver-rl8kx   1/1       Running   0          46s
```

**Step 5** Our replication controller created two new pods (replicas). The existing pod echoserver does not have the label app: echoserver, therefore it is not controlled by our replication controller. Let’s add this label the echoserver pod:

```
$ kubectl label pods echoserver app=echoserver
pod "echoserver" labeled
```

**Step 6** List pods:
```
$ kubectl get pods
NAME               READY     STATUS    RESTARTS   AGE
echoserver         1/1       Running   0          1h
echoserver-rl8kx   1/1       Running   0          1h
``` 

**Step 7**   Our replication controller has detected that there are three pods labeled with app: echoserver, so one pod has been stopped by the controller. Use kubectl describe to see controller events:
```
$ kubectl describe replicationcontroller/echoserver
Name:               echoserver
Namespace:  default
Image(s):   gcr.io/google_containers/echoserver:1.4
Selector:   app=echoserver
Labels:             app=echoserver
Replicas:   2 current / 2 desired
Pods Status:        2 Running / 0 Waiting / 0 Succeeded / 0 Failed
No volumes.
Events:
  ... Reason                        Message
  ... --------                      -------
  ... SuccessfulDelete      Deleted pod: echoserver-obzuw
```

**Step 8** To scale the number of replicas up we need to update the field replicas. Edit the file echoserver-rc.yaml, change the number of replicas to 3. Then use kubectl replace to update the replication controller:
```
$ kubectl replace -f echoserver-rc.yaml
replicationcontroller "echoserver" replaced
```

**Step 9** Use kubectl describe to check that the number of replicas has been updated in the controller:
```
$ kubectl describe replicationcontroller/echoserver
...
Replicas:   3 current / 3 desired
Pods Status:        3 Running / 0 Waiting / 0 Succeeded / 0 Failed
...
```
**Step 10** Let’s check the number of pods:
```
$ kubectl get pods
NAME               READY     STATUS    RESTARTS   AGE
echoserver         1/1       Running   0          2h
echoserver-b5ujn   1/1       Running   0          2m
echoserver-rl8kx   1/1       Running   0          1h
```
You can see that the replication controller has started a new pod.

###  Create a Service
We have three running echoserver pods, but we cannot use them from our lab, because the container ports are not accessible. Let’s define a new service that will expose echoserver ports and make them accessible from the VM.

**Step 1**  Create a new file echoserver-service.yaml with the following content:
```
apiVersion: v1
kind: Service
metadata:
  name: echoserver
spec:
  type: "NodePort"
  ports:
    - port: 8080
      protocol: TCP
      targetPort: 8080
  selector:
    app: echoserver
```

**Step 2** Create a new service:
```
$ kubectl create -f echoserver-service.yaml
service "echoserver" created
```

**Step 3** Check the service details:
```
$ kubectl describe services/echoserver
Name:               echoserver
Namespace:          default
Labels:             <none>
Selector:           app=echoserver
Type:               NodePort
IP:                 ...
Port:               <unset> 8080/TCP
NodePort:           <unset> 31698/TCP
Endpoints:          ...:8080,...:8080,..:8080
Session Affinity:   None
No events.
```
Note that the output contains three endpoints and a node port. It is 31698 in the output above, but it can be different in your case. Remember this port to use it in the next step.

**Step 4** To access a service exposed via a node port, specify the node port from the previous step:
```
$ curl http://localhost:<port>
CLIENT VALUES:
client_address=...
command=GET
real path=/
...
```

### Delete a Service, Controller, Pod

**Step 1** Before diving into Kubernetes deployment, let’s delete our service, controller, pods. To delete the service execute the following command:
```
$ kubectl delete service echoserver
service "echoserver" deleted
```

**Step 2** To delete the replication controller and its pods:
```
$ kubectl delete replicationcontroller echoserver
replicationcontroller "echoserver" deleted
```
**Step 3** Check that there are no running pods:
```
$ kubectl get pods
```

### Create a Deployment
**Step 1** The simplest way to create a new deployment for a single-container pod is to use kubectl run:
           
```
$ kubectl run echoserver \
--image=gcr.io/google_containers/echoserver:1.4 \
--port=8080 \
--replicas=2
deployment "echoserver" created
```

**Step 2** Check pods:
           
```
$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
echoserver-722388366-5z1bh   1/1       Running   0          1m
echoserver-722388366-nhfb1   1/1       Running   0          1m
```

**Step 3**  To access the echoserver from the lab, create a new service using kubectl expose deployment:
```
$ kubectl expose deployment echoserver --type=NodePort
service "echoserver" exposed
```
To get the exposed port number execute:
```
$ kubectl describe services/echoserver | grep ^NodePort
NodePort:           <unset> 30512/TCP
```
Remember the port number for the next step.

**Step 4**  Check that the echoserver is accessible:
```
$ curl http://localhost:<port>
CLIENT VALUES:
...
```
**Step 5**  Let’s change the number of replicas in the deployment. Use kubectl edit to open an editor and change the number of replicas to 3:
```
$ kubectl edit deployment echoserver
# edit the deployment definition, change replicas to 3
deployment "echoserver" edited
```

**Step 6** View the deployment details:
```
$ kubectl describe deployment echoserver
Name:                   echoserver
Namespace:              default
Labels:                 run=echoserver
Selector:               run=echoserver
Replicas:               3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
OldReplicaSets:                 <none>
NewReplicaSet:                  echoserver-722388366 (3/3 replicas created)
...
```

**Step 7** Check that there are 3 running pods:
           
```
$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
echoserver-722388366-5p5ja   1/1       Running   0          1m
echoserver-722388366-5z1bh   1/1       Running   0          1h
echoserver-722388366-nhfb1   1/1       Running   0          1h
```
**Step 8** Use kubectl rollout history deployment to see revisions of the deployment:
```
$ kubectl rollout history deployment echoserver
deployments "echoserver":
REVISION    CHANGE-CAUSE
1           <none>
```

**Step 9** Now we want to replace our echoserver with a new implementation. We want to use a new image based on busybox. Edit the deployment:
```
$ kubectl edit deployment echoserver
```

**Step 10** Change the image value to:
```
image: gcr.io/google-containers/busybox
```
**Step 11**  And add a new command field just after the image:
```
image: gcr.io/google-containers/busybox
command: ['nc', '-p', '8080', '-l', '-l', '-e', 'echo', 'hello world!']
```

**Step 12** Check the deployment status:
```
$ kubectl describe deployment echoserver
...
Replicas:           3 updated | 3 total | 3 available | 0 unavailable
...
```

**Step 13** Check that the echoserver works (use the port number from the step 3):           
```
$ curl http://localhost:<port>
hello world!
```

**Step 14** The deployment controller replaced all of the pods by new ones, one by one. Let’s check the revisions:
            
```
$ kubectl rollout history deployment echoserver
deployments "echoserver":
REVISION    CHANGE-CAUSE
1           <none>
2           <none>
```
**Step 15**   After that, we decided that the new implementation does not work as expected (we wanted echoserver, not a hello world application). Let’s undo the last change:

```
$ kubectl rollout undo deployment echoserver
deployment "echoserver" rolled back
```

**Step 16** Check the deployment status:          
```
$ kubectl describe deployment echoserver
...
Replicas:           3 updated | 3 total | 3 available | 0 unavailable
...
```

**Step 17**  Check that the echoserver works (use the port number from the step 3):
```
$ curl http://localhost:<port>
CLIENT VALUES:
```

**Step 18** Delete the deployment:
```
$ kubectl delete deployment echoserver
deployment "echoserver" deleted
```
We have successfully rolled back the deployment and our pods are based on the echoserver image again.

### Create a Job

**Step 1** Define a new job in the file myjob.yaml:
```
apiVersion: batch/v1
kind: Job
metadata:
  name: myjob
spec:
  completions: 5
  parallelism: 1
  template:
    metadata:
      name: myjob
    spec:
      containers:
      - name: myjob
        image: busybox
        command: ["sleep", "20"]
      restartPolicy: Never
```
This job is based on the image busybox. It waits 20 seconds, then exists. We requested 5 successful completions with no parallelism.


**Step 2** Create a new job:
```
$ kubectl create -f myjob.yaml
job "myjob" created
```
**Step 3** Let’s watch the job being executed and the results of each execution:

```
$ kubectl get jobs --watch
NAME      DESIRED   SUCCESSFUL   AGE
myjob     5         1            1m
...
```

**Step 4** If we’re interested in more details about the job:
```
$ kubectl describe jobs myjob
...
Pods Statuses:      1 Running / 0 Succeeded / 0 Failed
...
```


**Step 5** And finally:
```
$ kubectl describe jobs myjob
...
Pods Statuses:      0 Running / 5 Succeeded / 0 Failed
...
```
**Step 6**  After that, the job can be deleted:
```
$ kubectl delete job myjob
job "myjob" deleted
```