# Learn Kubernetes

## Course Notes

This README contains the notes I took when going through the [Boot.dev](https://www.boot.dev) course [Learn Kubernetes](https://www.boot.dev/courses/learn-kubernetes).

The files in this project are the various config files I created while doing the exercises contained in the course. The notes, without having done the exercises, is of minimal value. They are for myself, and they are not a representation of all topics and details presented in the course. If you are a software developer interested in learning Kubernetes, consider signing up for [Boot.dev](https://www.boot.dev). In my mind opinion, boot.dev offers some of the best software development training for the money.

### Chapter 1 - Install

In this course we'll be using [Kubernetes](https://kubernetes.io) with [Docker](https://www.docker.com). Will also be using [Minikube](https://minikube.sigs.k8s.io/docs/). Minikube is not for production deployments. Rather, it allows a developer to run a local cluster for the purpose of learning Kubernetes. Ensure Docker daemon is running before starting Minikube.

[Install kubernetes](https://kubernetes.io/docs/tasks/tools/) command-line tool kubectl. For MacOS, 

```brew install kubectl```

```kubectl version --client``` 

For course, install Minikube so we can simulate multiple servers on computer

```brew install minikube```

```minikube version```

Run:

```
minikube start --extra-config "apiserver.cors-allowed-origins=["http://boot.dev"]"
```
The extra configuration in the command above is to allow boot.dev to access the minikube cluster for exercise verification purposes.

Run the Minikube dashboard:

```
minikube dashboard --port=63840
```

This course doesn't use the minikube dashboard in the course, but it is a good resource to view and manage our Kubernetes cluster. The port is set to 63840 because that is the port the [Bootdev.cli tool](https://github.com/bootdotdev/bootdev) checks to validate course assignments. The course expects that minikube will be running throughout the time the student is doing the course.

The course is centered around to setting up the app [SynergyChat](https://github.com/bootdotdev/synergychat) to be deployed on Kubernetes.

Deploy a container built by boot.dev for this course:

`kubectl create deployment` creates a Kubernetes deployment. The minimum things that must be provided are the name of the deployment (can be anything) and the ID of the Docker image to deploy. Run the command below to create a _SynergyChat_ deployment. The command will deploy a container built from the [boot.dev SynergyChat-web docker image](https://hub.docker.com/search?q=bootdotdev).


```
kubectl create deployment synergychat-web --image=docker.io/bootdotdev/synergychat-web:latest
```

View (and verify) deployment:

```
kubectl get deployments
```

By default, resources inside a Kubernetes cluster run on a private, isolated, network not visible outside the cluster. To access the application from one's local network, port forwarding is required.

First, view the pods that were created when the `kubectl create deployment` command was run:

```
kubectl get pods
```

A pod is an abstraction over a container. Oversimplified, a pod is a running application.

Since resources inside a K8 cluster are only accessible inside the cluster, we need to do some port forwarding to be able to access our K8 cluster (a single pod, for now) within the local network. PODNAME is the name of the pod you see when you run `kubectl get pods`. The pod name I see at the time I am writing these notes is `synergychat-web-f765d99db-8h6rh`.

Run:
```
kubectl port-forward PODNAME 8080:8080
```

Open browser to http://localhost:8080 and should see webpage called SynergyChat

Note that using `kubectl port-forward` is not how access to pods happens in production. This is typically a technique used for development tasks, debugging, testing, temporarily connecting to a db, reaching services not meant to be publicly exposed. Production services rely on Services, Ingress Controllers, LoadBalanceer Services, NodePort Services, all of which are discussed later. Port-forwarding only works while the kubectl command is running and is not load balanced across multiple pods (it also, obviously, requires kubectl access).

Minikube supports only a single node, so you don't get the real benefits of K8 (Kubernetes). It's great for learning, but not for real-world deployments.

You can find some K8 case studies [here](https://www.cncf.io/case-studies/). [This one](https://www.cncf.io/case-studies/bloomberg/) from Bloomberg shows their experience deploying hundreds of clusters running thousands of nodes each.

### Chapeter 2 -- Pods

Use `kubectl get pods` to see list of all running pods. Should just see one synergychat-web pod at this point in the exercise. Let's add a second instance.

```
kubectl edit deployment synergychat-web
```

Under the `spec` section, change `replicas` from 1 to 2

```
spec:
  ...
  replicas: 2
  ...
```

Run `kubectl get pods` again and should see two pods.

To see what the containers are printing to stdout:

```
kubectl logs PODNAME
```

Kill the _older_ pid:

```
kubectl delete pod PODNAME
```

Get list of running pods again with `kubectl get pods` and there should be two pods. Because the spec dictates 2 replicas, kubernetes spins up a new pod when one dies.

Every pod in a kubernetes cluster has a unique internal-to-k8s IP address, which simplifies communication and service discovery within the cluster. Pods within the same node or across nodes can easily communicate. All resources are virtualized, so the IP address of a pod is not the same as the IP address of the node on which it is running. Instead, it is a virtual IP address only accessible within the cluster.

Run the following command to get a _wide_ output of the pods:

```
kubectl get pods -o wide
```

That command gives a few more columns of information, including the virtual IP address. Notice that each pod has a unique IP address.

Next, run:

```
kubectl proxy
```

That will start a proxy server on the local machine, most likely on 127.0.0.1:8001. Assuming that is the host, open a browser to `http://127.0.0.1:8001/api/v1/namespace/default/pods`. You should see a large JSON block that describes the pods that are running.

### Chapter 3 - Deployments

A [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) provides declarative updates for pods and replica sets.

You describe the desired state in a deployment and the deployment controller's task is to ensure the current state matches the desired state. That is what caused a new pod to be started after stopping one for the synergychat-web pods earlier.

Look at the YAML file for the current deployment in the CLI:

```
kubectl get deployment synergychat-web -o yaml
```

Edit the deployment to change the number of replicas from 2 to 10:

```
kubectl edit deployment synergychat-web
```

Now, ensure that 10 pods are running:

```
kubectl get pod
```

Once all 10 pods are in a ready state, run:

```
kubectl proxy
```

Now, viewing the contents of the url below (in browser or via curl) should show the deployment data for the synergychat-web app:

```
curl http://127.0.0.1:8001/apis/apps/v1/namespaces/default/deployments/synergychat-web
```

#### Replica Sets

A replica set maintains a stable set of replica pods. Replica sets are managed by deployments so you usually don't need to deal with them directly, but they may be referenced in logs.

#### Look at Replica Sets

```
kubectl get replicasets
```

Note: we never directly created a replica set. We created a deployment and the deployment created the replica set.

#### YAML Config

Kubernetes resources are primarily configured using YAML files.

Download a copy of your deployment's YAML file and save it in your current directory:

```
kubectl get deployment synergychat-web -o yaml > web-deployment.yaml
```

There are 5 top-level fields:

- `apiVersion: apps/v1` - Specifies the version of the Kubernetes API
- `kind: Deployment` - Specifies the type of object being configured
- `metadata` - Information about the the deployment. E.g. Date/time created, Name, ID.
- `spec` - The desired state of the deployment. Most impactful edits, like how many replicas desired, are made here
- `status` - Current state of the deployment. This section is not edited directly.

Open the web-deployment.yaml file in your preferred editor and change the number of replicas to 3. To apply the changes, run:

```
kubectl apply -f web-deployment.yaml
```

There should be warning alerting you that the `last-applied-configuration` annotation is missing. That is okay. In this case, the warning is because we created the deployment by running `kubectl create deployment` instead of by creating a YAML file and running `kubectl apply -f`, the later being the preferred method. But now that we've run `kubectl apply`, the annotation is added and the warning shouldn't be displayed again.

Download the YAML file again. The annotation should now be present. Apply the configuration again and you should not see the warning. _Save this YAML file in a git repo for this course. We'll be making more configuration files. Kubernetes is an "infra-as-code" tool, so it is important to keep the configuration files in a git repo._

Start the proxy server if not already running:

```
kubectl proxy
```

Note: like port forwarding, `kubectl proxy` is not something that is used in production, except for administrative tasks (sometimes). It is used primarily as a development and debugging tool. Production techniques for access use techniques like:

*Direct API server access with proper authentication*:

- Service accounts with RBAC
- Client certificates
- Bearer tokens
- OIDC integration

**API gateways** for external access to cluster services

**Ingress controllers** with proper TLS termination

**Service mesh** (Istio, Linkerd) for service-to-service communication

#### API Service

We've deployed multiple instances of the SynergyChat-web service. Now let's deploy a second service. 

We will deploy a JSON API service. It is the backend for the synergychat application. By deploying the API and configuring the frontend to talk to it, we'll have created a functional chat app.

Write the deployment file from scratch. Reference the [K8 docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment) as needed for proper structure.

See the requirements to create the file in the [Boot.dev Ch3, Lesson 5 Description](https://www.boot.dev/lessons/4643aa6b-ede1-4f01-ab4c-9a05763ee0ef) for the deployment configuration required.

Here is the file I created:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: synergychat-api
  name: synergychat-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: synergychat-api
  template:
    metadata:
      labels:
        app: synergychat-api
    spec:
      containers:
        - image: bootdotdev/synergychat-api:latest
          name: synergychat-api
```
#### Create the Deployment

```
kubectl apply -f api-deployment.yaml
```

Run the proxy so we can access the api pod (if not already running):

```
kubectl proxy
```

Now view the data either in the browser or in the terminal:

```
curl http://localhost:127.0.0.1:8001/apis/apps/v1/namespaces/default/deployments/synergychat-api
```

You should see status code 200, `.kind` equal to `Deployment`, and JSON at `.status.unavailableReplicas` to be equal to 1

Running

```
kubectl get pods
```

Should show that the SynergyChat-api service is in `CrashLoopBackoff` state

#### Thrashing Pods

Pods that keep crashing and restarting and called thrashing pods and are usually caused by one of a few things:

- The application had a bug introduced in the latest image version
- The application is misconfigured and cannot start properly
- A dependency of the application is misconfigured and the application cannot start properly
- The application is attempting to use excessive memory and is being killed by Kubernetes

#### What is "CrashLoopBackoff"?

When a pod's status is `CrashLoopBackoff`, that means the container is crashing (the program is exiting with error code 1)

Kubernetes will wait longer and longer each time is tries to restart a pod in CrashLoopBackoff state.

To fix a pod in this state, you (of course) must figure out why it is crashing.

### Chapter 4 - Config Maps

One of the most common ways to manage environment variables in Kubernetes is via the use of [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/). ConfigMaps allow decoupling configuration from container image. 

In a Dockerfile one can set a port like this:

```
ENV PORT=3000
```

But everyone that uses the container must use the same port. Changing the port requires rebuilding the image.

The synergychat-api deployment resulted in a crashing pod. To investigate, start by viewing the logs for the crashing pod:

```
kubectl get pods
kubectl logs <pod-name>
```

The logs for the synergychat-api pod shows that an environment variable is missing.

Let's fix that by creating a ConfigMap. Create a new file named api-configmap.yaml and add the foolowing YAML to it:

- apiVersion: v1
- kind: ConfigMap
- metadata/name: synergychat-api-configmap
- data/API_PORT: "8080":

Apply the config map:

```
kubectl apply -f api-configmap.yaml
```
We haven't yet connected the config map to the pod, so the pod should still be crashing. For now, validate that the config map was created successfully:

```
kubectl get configmaps
```

With `kubectl proxy` running, view the config maps url (browser or curl):

```
curl http://127.0.0.1:8001/api/v1/namespaces/default/configmaps/synergychat-api-configmap
```

You should see a status code 200, JSON `.kind` set to `ConfigMap`, and JSON `.data.API_PORT` equal to `8080`

#### Apply the Config Map

Open `api-deployment.yaml` and add the following under the `containers` section following the first (and only) entry:

```
env:
  - name: API_PORT
    valueFrom:
      configMapKeyRef:
        name: synergychat-api-configmap
        key: API_PORT
```

Here is what my api-deployment.yaml file looked like after completing this task:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: synergychat-api
  labels:
    app: synergychat-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: synergychat-api
  template:
    metadata:
      labels:
        app: synergychat-api
    spec:
      containers:
        - name: synergychat-api
          image: bootdotdev/synergychat-api:latest 
          env:
            - name: API_PORT
              valueFrom:
                configMapKeyRef:
                  name: synergychat-api-configmap
                  key:  API_PORT
```

With this config, Kubernetes will set the API_PORT environment variable to the value in API_PORT in the synergychat-api-configmap.

Apply the deployment using the same method used previously for applying config.

Forward the API pod's 8080 port to the local machine so it can be tested (again, this is a dev/test process, not a prod process):

```
kubectl port-forward <pod-name> 8080:8080
```

curl the endpoint:
```
curl http://localhost:8080
```
It should return a 404 right now.

Note that config maps are insecure and sensitive information should not be stored in them. Anyone with access to the cluster can access the config maps.

#### Crawler

The last application to deploy in this K8 cluster is the crawler. The crawler crawls [Project Gutenberg](https://www.gutenberg.org/) and exposes its data via a JSON API.

##### Add a New Config Map

Copy `api-configmap.yaml` to `crawler-configmap.yaml` and make the following changes:

- Name it `synergychat-crawler-configmap`
- Remove the `API_PORT` environment variable
- Add the following environment variables
    - `CRAWLER_PORT`: "8080"
    - `CRAWLER_KEYWORDS: love,hate,joy,sadness,anger,disgust,fear,surprise`

Deploy the config map:

```
kubectl apply -f crawler-configmap.yaml
```

##### Add a New Deployment

Maybe we got a bit ahead of ourselves by creating the crawler config map before creating the crawler deployment. Let's create the crawler deployment.

Copy `api-deployment.yaml` to `crawler-deployment.yaml` and make the following changes:

- Update all `synergychat-api` references to `synergychat-crawler`
- Update the image URL to `bootdotdev/synergychat-crawler:lastest`
- Update the environment variable references to match the new config map. 

Note: using
```
...
    env:
      - name: THING_ONE
        valueFrom:
          configMapKeyRef:
            name: synergychat-api-configmap
            key: THING_ONE
      - name: THING_TWO
        valueFrom:
          configMapKeyRef:
            name: synergychat-api-configmap
            key: THING_TWO
```
can be redundant. Let's use the `envFrom` key instead of copying each key from the config map:

```
envFrom:
  - configMapRef:
      name: synergychat-crawler-configmap
```

Apply it:

```
kubectl apply -f crawler-deployment.yaml
```

Here is what my `crawler-deployment.yaml` file looked like after completing this task:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: synergychat-crawler
  labels:
    app: synergychat-crawler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: synergychat-crawler
  template:
    metadata:
      labels:
        app: synergychat-crawler
    spec:
      containers:
        - name: synergychat-crawler
          image: bootdotdev/synergychat-crawler:latest 
          envFrom:
            - configMapRef:
                name: synergychat-crawler-configmap
```

Check that the pod is ready. If not, and the error is related to environment variables, debug your config map and deployment files and reapply them.

Once it is ready, forward the pod's 8080 port to your local machine for testing:

```
kubectl port-forward <pod-name> 8080:8080
```

Test the endpoint. We should see status code 200

```
curl -i http://localhost:8080/healthz
```

### Chapter 5 - Services

It isn't practical to spin up pods and connect to them individually, as we have been doing. Instead, we will switch to use services. [Services](https://kubernetes.io/docs/concepts/services-networking/service/) provide a stable endpoint for pods. They are an abstraction used to provide a stable endpoint and load balance traffic across a group of pods. "Stable endpoint" in this context simply means that the service will always be available at a given URL even if the pod is destroyed and recreated. A network request goes to the service which in turns forwards it to the pod(s).

![image depicting how services function in a kubernetes environment](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/7JCPRd3-1280x717.png)
(image property of [boot.dev](https://www.boot.dev))

Let's add a service for the 3 synergycat-web pods. Create a file called web-service.yaml and add the following:

- `apiVersion: v1`
- `kind`: `Service`
- `metadata/name`: `web-service`
- `spec/selector/app`: `synergychat-app`
- `spec/ports`: An array of port objects. One entry required:
    - `protocol`: `TCP` ([TCP will allow the use of HTTP](https://www.quora.com/What-is-the-difference-between-HTTP-protocol-and-TCP-protocol/answer/Daniel-Miller-7?ch=10&oid=3824340&share=340dfe9e&srid=iRqdc&target_type=answer))
    - `port`: `8080` (this is the port the pods are listening on)

This creates a new service called `web-service` with the following properties: 

- Listens for incoming traffic on port `80`
- Forwards traffic to synergychat-web pods on port `8080`
- The controller for the service continually scans for pods with the `app:synergychat-web` label selector and adds them to its pool.

Create the service by applying the web-service.yaml file:

```
kubectl apply -f web-service.yaml
```

Forward the service's ports to local machine to test:

```
kubectl port-forward service/web-service 8080:80
```

Web app should now be available at `http://localhost:8080`

#### Service Types

There are several types of service. The default is ClusterIP. If you run:

```
kubectl get service web-service -o yaml
```

You'll see a section that shows:

```
spec:
  clusterIP: <ip address>
  ...
  type: ClusterIP
```

The `clusterIP` is the IP address the service is bound to on the internal Kubernetes network. There are several other types of service, including:

- NodePort
- LoadBalancer
- ExternalName

Service types are documented [here](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)

##### API Service

Let's create an api service which will be responsible for handling requests from the front end and returning data from the db. Make the api service type "NodePort" to expose the api service to the outside world.

Copy web-service.yaml to api-service.yaml and make the following edits:

- Name should be `api-service`
- Should select pods using the `app: synergychat-api` key
- Add `type: NodePort` to the `spec` section
- add `nodePort: 30080` to the first object in the ports list

Docs for NodePort service are [here](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)

My file looks like this:
```
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: NodePort
  selector:
    app: synergychat-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30080
```
Apply the service and then check to ensure it is running:

```
kubectl get svc
```

(note: kubectl allows use of either `svc` or `service`)

If not already running `kubectl proxy`, do so. 

```
curl http://127.0.0.1:8001/api/v1/namespaces/default/asp-service
```


### Chapter 6 - Ingress

An [ingress](https://kubernetes.io/docs/concepts/services-networking/ingress) resource exposes services to the outside world and is used often in production environments. From the docs:


> An Ingress may be configured to give Services externally-reachable URLs, load balance traffic, terminate SSL / TLS, and offer name-based virtual hosting. An Ingress controller is responsible for fulfilling the Ingress, usually with a load balancer, though it may also configure your edge router or additional frontends to help handle the traffic.
>
> An Ingress does not expose arbitrary ports or protocols. Exposing services other than HTTP and HTTPS to the internet typically uses a service of type Service.Type=NodePort or Service.Type=LoadBalancer.


![image showing ingress functionality](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/4Bu6KOe-826x400.png)
(image property of [Boot.dev](https://www.boot.dev))
