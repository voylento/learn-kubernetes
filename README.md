# Learn Kubernetes

## Course Notes

This README contains the notes I took when going through the [Boot.dev](https://www.boot.dev) course [Learn Kubernetes](https://www.boot.dev/courses/learn-kubernetes).

The files in this project are the various config files I created while doing the exercises contained in the course. The notes, without having done the exercises, is of minimal value. They are for myself, and they are not a representation of all topics and details presented in the course. If you find my notes of any value, consider signing up for [Boot.dev](https://www.boot.dev). In my mind opinion, boot.dev offers some of the best software development training for the money.

### Chapter 1 - Install

In this course we'll be using Kubernetes with Docker. Ensure Docker daemon is running before starting Minikube.

Install kubernetes command-line tool kubectl. For MacOS, 

```brew install kubectl```

```kubectl version --client``` 

For course, install Minikube so we can simulate multiple servers on computer

```brew install minikube```

```minikube version```

Run:

```
minikube start --extra-config "apiserver.cors-allowed-origins=["http://boot.dev"]"
```

Run the dashboard:

```
minikube dashboard --port=63840
```

Deploy a container built by boot.dev for this course:

```
kubectl create deployment synergychat-web --image=docker.io/bootdotdev/synergychat-web:latest
```

Verify deployment:

```
kubectl get deployments
```

Verify pods:

```
kubectl get pods
```

Run:
```
kubectl port-forward PODNAME 8080:8080
```

Open browser to http://localhost:8080 and should see webpage called SynergyChat


### Chapeter 2 -- Pods

Use `kubectl get pods` to see list of all running pods. Shoul just see one synergychat-web pod. Let's add a second instance.

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

#### Replica Sets

A replica set maintains a stable set of replica pods. Replica sets are managed by deployments so you usually don't need to deal with them directly, but they may be referenced in logs.

#### Look at Replica Sets

```
kubectl get replicasets
```

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

Kill the proxy server:

```
kubectl proxy
```

#### API Service

Now we will deploy a JSON API service. It is the backend for the synergychat application. By deploying the API and configuring the frontend to talk to it, we'll have created a working chat app.

Write the deployment file from scratch. Reference the [K8 docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment) as needed for proper structure.

See the requirements to create the file in the [Boot.dev Ch3, Lesson 5 Description](https://www.boot.dev/lessons/4643aa6b-ede1-4f01-ab4c-9a05763ee0ef) for the deployment configuration required.

#### Create the Deployment

```
kubectl apply -f api-deployment.yaml
```

Run:

```
kubectl proxy
```

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

Let's fix that by creating a ConfigMap. Create a new file names api-configmap.yaml and add the foolowing YAML to it:

- apiVersion: v1
- kind: ConfigMap
- metadata/name: synergychat-api-configmap
- data/API_PORT: "8080:

Apply the config map:

```
kubectl apply -f api-configmap.yaml
```
We haven't yet connected the config map to the pod, so the pod should still be crashing. For now, validate that the config map was created successfully:

```
kubectl get configmaps
```

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

Forward the API pod's 8080 port to the local machine so it can be tested:

```
kubectl port-forward <pod-name> 8080:8080
```

It should return a 404 right now:

```
curl http://localhost:8080
```

Note that config maps are insecure and sensitive information should not be stored in them. Anyone with access to the cluster can access the config maps.

#### Crawler

Let's create a crawler to crawls [Project Gutenberg](https://www.gutenberg.org/) and exposes its data via a JSON API.

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

Copy `api-deployment.yaml` to `crawler-deployment.yaml` and make the following changes:

- Update all `synergychat-api` references to `synergychat-crawler`
- Update the image URL to `bootdotdev/synergychat-crawler:lastest`
- Update the environment variable references to match the new config map. Use the `envFrom` key instead of copying each key from the config map:

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

### Chapter 5 - Services

Services provide a stable endpoint for pods. They are an abstraction used to provide a stable endpoint and load balance traffic across a group of pods. "Stable endpoint" in this context simply means that the service will always be available at a given URL even if the pod is destroyed and recreated. A network request goes to the service which in turns forwards it to the pod(s).



