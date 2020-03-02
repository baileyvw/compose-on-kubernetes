# Compose on Kubernetes

[![CircleCI](https://circleci.com/gh/docker/compose-on-kubernetes/tree/master.svg?style=svg)](https://circleci.com/gh/docker/compose-on-kubernetes/tree/master)

Compose on Kubernetes allows you to deploy Docker Compose files onto a
Kubernetes cluster.

# Table of contents

- [Get started](#get-started)
- [Developing Compose on Kubernetes](#developing-compose-on-kubernetes)

More documentation can be found in the [docs/](./docs) directory. This includes:
- [Architecture](./docs/architecture.md)
- [Mapping of stack to Kubernetes objects](./docs/mapping.md)
- [Compatibility matrix](./docs/compatibility.md)

# Get started

Compose on Kubernetes comes installed on
[Docker Desktop](https://www.docker.com/products/docker-desktop) and
[Docker Enterprise](https://www.docker.com/products/docker-enterprise).

On Docker Desktop you will need to activate Kubernetes in the settings to use
Compose on Kubernetes.

## Check that Compose on Kubernetes is installed

You can check that Compose on Kubernetes is installed by checking for the
availability of the API using the command:

```console
$ kubectl api-versions | grep compose
compose.docker.com/v1beta1
compose.docker.com/v1beta2
```

## Deploy a stack

To deploy a stack, you can use the Docker CLI:

```console
$ cat docker-compose.yml
version: '3.3'

services:

  db:
    build: db
    image: dockersamples/k8s-wordsmith-db

  words:
    build: words
    image: dockersamples/k8s-wordsmith-api
    deploy:
      replicas: 5

  web:
    build: web
    image: dockersamples/k8s-wordsmith-web
    ports:
     - "33000:80"

$ docker stack deploy --orchestrator=kubernetes -c docker-compose.yml hellokube
```

## Remove a stack

```
$ docker stack rm --orchestrator=kubernetes hellokube
```

# Developing Compose on Kubernetes

See the [contributing](./CONTRIBUTING.md) guides for how to contribute code.

## Pre-requisites

- `make`
- [Docker Desktop](https://www.docker.com/products/docker-desktop) (Mac or Windows) with engine version 18.09 or later
- Enable Buildkit by setting `DOCKER_BUILDKIT=1` in your environment
- Enable Kubernetes in Docker Desktop settings

### For live debugging

- Debugger capable of remote debugging with Delve API version 2
  - Goland run-configs are pre-configured

## Debug quick start

### Debug install

To build and install a debug version of Compose on Kubernetes onto Docker
Desktop, you can use the following command:

```console
$ make -f debug.Makefile install-debug-images
```

This command:
- Builds the images with debug symbols
- Runs the debug installer:
  - Installs debug versions of API server and Compose controller in the `docker` namespace
  - Creates two debugging _LoadBalancer_ services (unused in this mode)

You can verify that Compose on Kubernetes is running with `kubectl` as follows:

```console
$ kubectl get all -n docker
NAME                               READY   STATUS    RESTARTS   AGE
pod/compose-7c4dfcff76-jgwst       1/1     Running   0          59s
pod/compose-api-759f8dbb4b-2z5n2   2/2     Running   0          59s

NAME                                      TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE
service/compose-api                       ClusterIP      10.98.42.151     <none>        443/TCP           59s
service/compose-api-server-remote-debug   LoadBalancer   10.101.198.179   localhost     40001:31693/TCP   59s
service/compose-controller-remote-debug   LoadBalancer   10.101.158.160   localhost     40000:31167/TCP   59s

NAME                          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/compose       1         1         1            1           59s
deployment.apps/compose-api   1         1         1            1           59s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/compose-7c4dfcff76       1         1         1       59s
replicaset.apps/compose-api-759f8dbb4b   1         1         1       59s
```

If you describe one of the deployments, you should see `*-debug:latest` in the
image name.

### Live debugging install

To build and install a live debugging version of Compose on Kubernetes onto
Docker Desktop, you can use the following command:

```console
$ make -f debug.Makefile install-live-debug-images
```

This command:
- Builds the images with debug symbols
- Sets the image entrypoint to run a [Delve server](https://github.com/derekparker/delve)
- Runs the debug installer
  - Installs debug version of API server and Compose controller in the `docker` namespace
  - Creates two debugging _LoadBalancer_ services
    - `localhost:40000`: Compose controller
    - `localhost:40001`: API server
- The API server and Compose controller only start once a debugger is attached

To attach a debugger you have multiple options:
- Use [GoLand](https://www.jetbrains.com/go/): configuration can be found in `.idea` of the repository
  - Select the `Debug all` config, setup breakpoints and start the debugger
- Set your Delve compatible debugger to point to use `locahost:40000` and `localhost:40001`
  - Using a terminal: `dlv connect localhost:40000` then type `continue` and hit enter

To verify that the components are installed, you can use the following command:

```console
$ kubectl get all -n docker
```

To verify that the API server has started, ensure that it has started logging:
```console
$ kubectl logs -f -n docker deployment.apps/compose-api compose
API server listening at: [::]:40000
ERROR: logging before flag.Parse: I1207 15:25:13.760739      11 plugins.go:158] Loaded 2 mutating admission controller(s) successfully in the following order: NamespaceLifecycle,MutatingAdmissionWebhook.
ERROR: logging before flag.Parse: I1207 15:25:13.763211      11 plugins.go:161] Loaded 1 validating admission controller(s) successfully in the following order: ValidatingAdmissionWebhook.
ERROR: logging before flag.Parse: W1207 15:25:13.767429      11 client_config.go:552] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
ERROR: logging before flag.Parse: W1207 15:25:13.851500      11 genericapiserver.go:319] Skipping API compose.docker.com/storage because it has no resources.
ERROR: logging before flag.Parse: I1207 15:25:13.998154      11 serve.go:116] Serving securely on [::]:9443
```

To verify that the Compose controller has started, ensure that it is logging:
```console
kubectl logs -f -n docker deployment.apps/compose
API server listening at: [::]:40000
Version:    v0.4.16-dirty
Git commit: b2e3a6b-dirty
OS/Arch:    linux/amd64
Built:      Fri Dec  7 15:18:13 2018
time="2018-12-07T15:25:19Z" level=info msg="Controller ready"
```

## Reinstall default

To reinstall the default Compose on Kubernetes on Docker Desktop, simply restart
your Kubernetes cluster. You can do this by deactivating and then reactivating
Kubernetes or by restarting Docker Desktop.
See the [contributing](./CONTRIBUTING.md) and [debugging](./DEBUGGING.md) guides.

# Deploying Compose on Kubernetes

- Guide for [Azure AKS](./docs/install-on-aks.md).
- Guide for [GKE](./docs/install-on-gke.md).
- Guide for [Microk8s](./docs/install-on-microk8s.md).
- Guide for [Minikube](./docs/install-on-minikube.md).

Why do I need Compose if I already have Kubernetes?

The Kubernetes API is really quite large. There are more than 50 first-class objects in the latest release, from Pods and Deployments to ValidatingWebhookConfiguration and ResourceQuota. This can lead to a verbosity in configuration, which then needs to be managed by you, the developer. Let’s look at a concrete example of that.

The Sock Shop is the canonical example of a microservices application. It consists of multiple services using different technologies and backends, all packaged up as Docker images. It also provides example configurations using different tools, including both Compose and raw Kubernetes configuration. Let’s have a look at the relative sizes of those configurations:

$ git clone https://github.com/microservices-demo/microservices-demo.git
$ cd deployment/kubernetes/manifests
$ (Get-ChildItem -Recurse -File | Get-Content | Measure-Object -line).Lines
908
$ cd ../../docker-compose
$ (Get-Content docker-compose.yml | Measure-Object -line).Lines
174
Describing the exact same multi-service application using just the raw Kubernetes objects takes more than 5 times the amount of configuration than with Compose. That’s not just an upfront cost to author – it’s also an ongoing cost to maintain. The Kubernetes API is amazingly general purpose – it exposes low-level primitives for building the full range of distributed systems. Compose meanwhile isn’t an API but a high-level tool focused on developer productivity. That’s why combining them together makes sense. For the common case of a set of interconnected web services, Compose provides an abstraction that simplifies Kubernetes configuration. For everything else you can still drop down to the raw Kubernetes API primitives. Let’s see all that in action.

First we need to install the Compose on Kubernetes controller into your Kubernetes cluster. This controller uses the standard Kubernetes extension points to introduce the `Stack` to the Kubernetes API. You can use any Kubernetes cluster you like, but if you don’t already have one available then remember that Docker Desktop comes with Kubernetes and the Compose controller built-in, and enabling it is as simple as ticking a box in the settings.

To install the controller manually on any Kubernetes cluster, see the full documentation for the current installation instructions.

Next let’s write a simple Compose file:

version: "3.7"
services:
  web:
    image: dockerdemos/lab-web
    ports:
     - "33000:80"
  words:
    image: dockerdemos/lab-words
    deploy:
      replicas: 3
      endpoint_mode: dnsrr
  db:
    image: dockerdemos/lab-db
We’ll then use the docker client to deploy this to a Kubernetes cluster running the controller:

$ docker stack deploy --orchestrator=kubernetes -c docker-compose.yml words
Waiting for the stack to be stable and running...
db: Ready       [pod status: 1/1 ready, 0/1 pending, 0/1 failed]
web: Ready      [pod status: 1/1 ready, 0/1 pending, 0/1 failed]
words: Ready    [pod status: 1/3 ready, 2/3 pending, 0/3 failed]
Stack words is stable and running
We can then interact with those objects via the Kubernetes API. Here you can see we’ve created the lower-level objects like Services, Pods, Deployments and ReplicaSets automatically:

$ kubectl get all
NAME                       READY     STATUS    RESTARTS   AGE
pod/db-85849797f6-bhpm8    1/1       Running   0          57s
pod/web-7974f485b7-j7nvt   1/1       Running   0          57s
pod/words-8fd6c974-44r4s   1/1       Running   0          57s
pod/words-8fd6c974-7c59p   1/1       Running   0          57s
pod/words-8fd6c974-zclh5   1/1       Running   0          57s

NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/db              ClusterIP      None            <none>        55555/TCP      57s
service/kubernetes      ClusterIP      10.96.0.1       <none>        443/TCP        4d
service/web             ClusterIP      None            <none>        55555/TCP      57s
service/web-published   LoadBalancer   10.102.236.49   localhost     33000:31910/TCP   57s
service/words           ClusterIP      None            <none>        55555/TCP      57s

NAME                    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/db      1         1         1            1           57s
deployment.apps/web     1         1         1            1           57s
deployment.apps/words   3         3         3            3           57s

NAME                             DESIRED   CURRENT   READY     AGE
replicaset.apps/db-85849797f6    1         1         1         57s
replicaset.apps/web-7974f485b7   1         1         1         57s
replicaset.apps/words-8fd6c974   3         3         3         57s
It’s important to note that this isn’t a one-time conversion. The Compose on Kubernetes API Server introduces the Stack resource to the Kubernetes API. So we can query and manage everything at the same level of abstraction as we’re building the application. That makes delving into the details above useful for understanding how things work, or debugging issues, but not required most of the time:

$ kubectl get stack
NAME      STATUS      PUBLISHED PORTS   PODS     AGE      
words     Running     33000             5/5      4m
Integration with other Kubernetes tools

Because Stack is now a native Kubernetes object, you can work with it using other Kubernetes tools. As an example, save the as `stack.yaml`:

kind: Stack
apiVersion: compose.docker.com/v1beta2
metadata:
 name: hello
spec:
 services:
 - name: hello
   image: garethr/skaffold-example
   ports:
   - mode: ingress
     target: 5678
     published: 5678
     protocol: tcp
You can use a tool like Skaffold to have the image automatically rebuild and the Stack automatically redeployed whenever you change any of the details of your application. This makes for a great local inner-loop development experience. The following `skaffold.yaml` configuration file is all you need.

apiVersion: skaffold/v1alpha5
kind: Config
build:
 tagPolicy:
   sha256: {}
 artifacts:
 - image: garethr/skaffold-example
 local:
   useBuildkit: true
deploy:
 kubectl:
   manifests:
     - stack.yaml
The future

We already have some thoughts about a Helm plugin to make describing your application with Compose and deploying with Helm as easy as possible. We have lots of other ideas for helping to simplify the developer experience of working with Kubernetes too, without losing any of the power of the platform. We also want to work with the wider Cloud Native community, so if you have ideas and suggestions please let us know.

Kubernetes is designed to be extended, and we hope you like what we’ve been able to release today. If you’re one of the millions of Compose users you can now more easily move to and manage your applications on Kubernetes. If you’re a Kubernetes user struggling with too much low-level configuration then give Compose a try. Let us know in the comments what you think, and head over to GitHub to try things out and even open your first PR:

Compose on Kubernetes controller

