# Deploying your Node.js App via Helm

## Packaging up your Node.js application into a Helm Chart

This will show you how to create your own Helm Chart for your Node.js Application and how to tweak it to best fit your applications needs.

In this self-paced tutorial you will:

- Create a Helm Chart
- Add a dependency chart
- Create liveness probes

The application you will use is the one created from - https://github.com/nodeshift/mern-workshop

## Prerequisites

Before getting started, make sure you have the following prerequisites installed on your system.

1. Install [Node.js 14](https://nodejs.org/en/download/) (or use [nvm](https://github.com/nvm-sh/nvm#installation-and-update))
1. Docker
   - **_On Mac_**: [Docker Desktop](https://docs.docker.com/desktop/mac/install/)
   - **_On Windows_**: [Docker Desktop](https://docs.docker.com/desktop/windows/install/)
   - **_On Linux_**: [Docker](https://docs.docker.com/desktop/windows/install/)
1. Kubernetes
   - **_On Mac_**: [Kubernetes](https://docs.docker.com/desktop/kubernetes/#enable-kubernetes)
   - **_On Windows_**: [Kubernetes](https://docs.docker.com/desktop/kubernetes/#enable-kubernetes)
   - **_On Linux_**: Docker Desktop is not available on linux, hence you can enable Kubernetes by choosing one of below alternatives:
     - [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)
     - [microk8s](https://microk8s.io/#quick-start)
1. Helm v3 - https://helm.sh/docs/intro/install/
   - **Note**: This workshop tested with Helm v3.4.0

## Setting up

### Docker

#### On linux

<details>
Install Docker Engine Community using

```sh
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

More information can be found at https://docs.docker.com/install/linux/docker-ce/ubuntu/

Add the user to the docker group (optional for part 1, required for part 2 and 3)

```sh
sudo groupadd docker
sudo gpasswd -a $USER docker
sudo service docker restart
```

Install docker compose

```sh
sudo apt install docker-compose
```

</details>

### Kubernetes

#### On macOS:

1. Select the Docker icon in the Menu Bar
2. Click `Preferences/Settings > Kubernetes > Enable Kubernetes`.

Ensure you have installed Docker Desktop and enabled Kubernetes within the application. To do so:

#### On Windows:

1. Select the Docker icon in the notification area of the taskbar.
2. Click `Settings > Kubernetes > Enable Kubernetes`.

It will take a few moments to install and start up. If you already use Kubernetes, ensure that you are configured to use the `docker-for-desktop` cluster. To do so:

1. Select the Docker icon in the Menu Bar
2. Click `Kubernetes` and select the `docker-for-desktop` context

#### On Linux:

You can install Kubernetes choosing **one** of below options:

#### minikube

https://minikube.sigs.k8s.io/docs/start

<details>

```sh
minikube start
eval $(minikube docker-env)
```

</details>

#### Microk8s

1. Install microk8s through snap 

    <details>
        <summary>Installation</summary>

    ```sh
    snap install microk8s --classic
    microk8s status --wait-ready
    microk8s enable dashboard dns registry istio
    ```
    for further details please visit: https://microk8s.io

    Enabling private registry in microk8s
    </details>

1. Enable pushing to insecure registry

    <details>
    
    Create file `/etc/docker/daemon.json` and add the following lines:


    ```
    {
        "insecure-registries" : ["localhost:32000"] 
    }
    ```

    restart docker

    ```
    sudo systemctl restart docker
    ```
    </details>


1. Enable private registry for Microk8s

    *Use `microk8s ctr version` command to fetch Microk8s version*

    <details>
        <summary>Version 1.23 or newer</summary>

    ```
    sudo mkdir -p /var/snap/microk8s/current/args/certs.d/localhost:32000
    sudo touch /var/snap/microk8s/current/args/certs.d/localhost:32000/hosts.toml
    ```
    Then, edit the file we just created and make sure the contents are as follows:

    ```
    # /var/snap/microk8s/current/args/certs.d/localhost:32000/hosts.toml
    server = "http://localhost:32000"

    [host."localhost:32000"]
    capabilities = ["pull", "resolve"]
    ```

    Restart MicroK8s to have the new configuration loaded:
    ```
    microk8s stop
    microk8s start
    ```

    for further details please visit: https://microk8s.io/docs/registry-private on the **For MicroK8s version 1.23 or newer** section
    </details>

    <details>
      <summary>Version 1.22 or older</summary>

      We need to edit `/var/snap/microk8s/current/args/containerd-template.toml` and add the following under `[plugins."io.containerd.grpc.v1.cri".registry.mirrors]`:

      ```
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."localhost:32000"]
      endpoint = ["http://localhost:32000"]
      ```

      Restart MicroK8s to have the new configuration loaded:

      ```
      microk8s stop
      microk8s start
      ```
    </details>

### Helm

Helm is a package manager for Kubernetes. By installing a Helm "chart" into your Kubernetes cluster you can quickly run all kinds of different applications. You can install Helm using one of the options below:

Choose one of the below section to install helm. For further indo please visit https://helm.sh/docs/intro/install/
#### Using a Package Manager:

- macOS with Homebrew: `brew install helm`
- Linux with Snap: `sudo snap install helm --classic`
- Windows with Chocolatey: `choco install kubernetes-helm`

#### Using microk8s:
If you installed microk8s, use below method to install helm

```sh
$ microk8s.enable helm3
```

#### Using a Script:

```sh
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

## Downloading the application

Make sure you clone down the application repo and you are in the correct directory.

```sh
$ git clone https://github.com/nodeshift/mern-workshop.git
$ cd mern-workshop
```

### 1. Create your Helm Chart files

In your terminal inside the folder holding your application run:

```sh
$ mkdir chart
$ cd chart
$ helm create myapp
```
or if you are using microk8s

```sh
$ mkdir chart
$ cd chart
$ microk8s helm3 create myapp
```

This will create the following file structure:

```
myapp/
├── .helmignore   # Contains patterns of which files to ignore when packaging your Helm Chart.
├── Chart.yaml    # Information about your chart
├── values.yaml   # The default values for your templates
├── charts/       # Charts that this chart depends on
└── templates/    # The template files
    └── tests/    # The test files
```

These files will form the basis of your Helm Chart, lets explore the pre-created files and make some necessary changes.

### 2. Editing the .helmignore file

This file works the same as any .ignore file, you just fill in the patterns you don't want to be packaged up into the helm chart. For example, if you have some secrets saved as a JSON file that you do not want to be inside the helm chart. For our application we do not have any files we need to protect so let's move on to the next step.

### 3. The Chart.yaml file

   This file contains the base information about your Helm Chart, your premade one should look similar to this (with extra comments):
   
   ```yaml
   apiVersion: v2
   name: myapp
   description: A Helm chart for Kubernetes
   type: application
   version: 0.1.0
   appVersion: 1.0.0
   ```

   `apiVersion: v2` signals that this chart is designed for Helm3 support _only_.

   `version` is the charts version, increment this under semver every time you update the chart

   `appVersion` is the version of the app you are deploying, this is to be increased every time you increase the version of your app but does not impact the charts version 

The rest of the fields are self explanatory but lets add some more information to describe our chart. We are going to set a Kubernetes minimum version, add some descriptive keywords and add our name as Maintainers. So go ahead and add the following to your `Chart.yaml` whilst subsituting your name and email in:

   ```yaml
   kubeVersion: '>= 1.21.0-0'
   keywords:   
   - nodejs
   - express
   - mern
   maintainers:
   - name: Firstname Lastname
     email: FirstnameLastname@company.com
   ```

The final key thing we are going to add is a dependency, our application needs mongoDB to run so we are going to call an existing mongo chart to install mongo as we install our chart. Firstly we need to add to our `Chart.yaml`:

   ```yaml
   dependencies:
   - name: mongodb
     version: 10.26.3
     repository: https://charts.bitnami.com/bitnami
   ```

   Then run the following command in the terminal to download the chart:
   
   ```sh
   helm dependency update
   ```
  or if you are using microk8s 
   ```sh
   microk8s helm3 dependency update
   ```

### 4. Template files

Inside the templates folder you will find that Helm has created some files for us already:

```sh
$ ls myapp/templates

NOTES.txt
_helpers.tpl
deployment.yaml
hpa.yaml
ingress.yaml
service.yaml
serviceaccount.yaml
tests
```

These files represent the kubernetes objects that our Helm Chart will deploy - you can set up the deployment and service, and you can also create a serviceaccount for your app to use. The `NOTES.txt` file is just the notes that will be displayed to the user when they deploy your Helm Chart, you can use this to provide further instruction to them to get your application working. `_helpers.tpl` is where template helpers that you can re-use throughout the chart go. However for this tutorial we are going to use our own files so you can go ahead and remove the generated files:

```sh
$ rm -rf chart/myapp/templates/*
```

Now let's move on to making our own template files!

#### 4.1 Frontend Service

The first template we will create will deploy the service for our frontend. Let's create a file called `chart/myapp/templates/frontend-service.yaml` and paste in the following:

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: 'true'
  name: "frontend-service"
spec:
  ports:
  - name: http
    port: {{ .Values.frontend.service.servicePort }}
    nodePort: 30444
  type: NodePort
  selector:
    app: "frontend-selector"
```

This will create the service that serves our frontend and allows for it to be connected to the outside world. `nodePort` is the port we want our service to be accessible through so `localhost:30444` will take us to our frontend once deployed

#### 4.2 Frontend Deployment

The next template we create will be the deployment for the frontend. Create a file called `chart/myapp/templates/frontend-deployment.yaml` and paste in the following:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  labels:
    chart: frontend
spec:
  selector:
    matchLabels:
      app: "frontend-selector"
  template:
    metadata:
      labels:
        app: "frontend-selector"
    spec:
      containers:
      - name: frontend
        image: "{{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}"
        imagePullPolicy: {{ .Values.frontend.image.pullPolicy }}
        livenessProbe:
          httpGet:
            path: /
            port: {{ .Values.frontend.service.servicePort }}
          initialDelaySeconds: {{ .Values.frontend.livenessProbe.initialDelaySeconds}}
          periodSeconds: {{ .Values.frontend.livenessProbe.periodSeconds}}
        ports:
        - containerPort: {{ .Values.frontend.service.servicePort}}
```

What this creates is our frontend deployment, inside that is the pod that is running our frontend image plus its replicaset.

`image` is the image we want the container to deploy using variables pulled from our `values.yaml` file.

We also add the `frontend-selector` label to our pod which allows for our frontend service to connect with it.

#### 4.3 Backend Service

Our third file will spawn the service for the backend part of our application. Create a file called `chart/myapp/templates/backend-service.yaml` and paste in the following:

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: 'true'
  name: "backend-service"
spec:
  ports:
  - name: http
    port: {{ .Values.backend.service.servicePort }}
    nodePort: 30555
  type: NodePort
  selector:
    app: "backend-selector"
```

This file does the same thing as `frontend-service.yaml` but for the backend and under a different port.

#### 4.4 Backend Deployment

Our final template is the deployment for the backend. Create a file called `chart/myapp/templates/backend-deployment.yaml` and paste in the following:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
  labels:
    chart: backend
spec:
  selector:
    matchLabels:
      app: "backend-selector"
  template:
    metadata:
      labels:
        app: "backend-selector"
    spec:
      containers:
      - name: backend
        image: "{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}"
        imagePullPolicy: {{ .Values.backend.image.pullPolicy }}
        livenessProbe:
          httpGet:
            path: /health
            port: {{ .Values.backend.service.servicePort }}
          initialDelaySeconds: {{ .Values.backend.livenessProbe.initialDelaySeconds}}
          periodSeconds: {{ .Values.backend.livenessProbe.periodSeconds}}
        ports:
        - containerPort: {{ .Values.backend.service.servicePort}}
        env:
          - name: PORT
            value : "{{ .Values.backend.service.servicePort }}"
          - name: APPLICATION_NAME
            value: "{{ .Release.Name }}"
          - name: MONGO_URL
            value: {{ .Values.backend.services.mongo.url }}
          - name: MONGO_DB_NAME
            value: {{ .Values.backend.services.mongo.name }}
```

Similar to our frontend deployment file, this file creates our backend deployment, with a key difference being the mongoDB information that is passed through to allow for communication with the mongo instance.

### 5. Values file

For the `chart/myapp/values.yaml` file we are going to split it into 3 sections, have a read of each section and then add them all to your `values.yaml` file

```yaml
# Frontend
frontend:
  replicaCount: 1
  revisionHistoryLimit: 1
  image:
    repository: frontend 
    tag: v1.0.0
    pullPolicy: IfNotPresent
    resources:
      requests:
        cpu: 200m
        memory: 300Mi
  livenessProbe:
    initialDelaySeconds: 30
    periodSeconds: 10
  service:
    name: frontend
    type: NodePort
    servicePort: 80 # the port where nginx serves its traffic
```
These values are for the frontend section. Here we pass through the image name, tag and the values for the frontend service.

In case of microk8s replace  `repository: frontend` with `repository: localhost:32000/frontend`

```yaml
# backend
backend:
  replicaCount: 1
  revisionHistoryLimit: 1
  image:
    repository: backend
    tag: v1.0.0
    pullPolicy: IfNotPresent
    resources:
      requests:
        cpu: 200m
        memory: 300Mi
  livenessProbe:
    initialDelaySeconds: 30
    periodSeconds: 10
  service:
    name: backend
    type: NodePort
    servicePort: 30555
  services:
    mongo:
      url: myapp-mongodb
      name: todos
      env: production
```

These values are for the backend section. Here we pass through the image, tag, service information, and some mongoDB information to locate the instance.

In case of microk8s replace  `repository: backend` with `repository: localhost:32000/backend`

```yaml
# mongo
mongodb:
  auth:
    enabled: false
  replicaSet:
    enabled: true
    replicas:
      secondary: 3
  service:
    type: LoadBalancer

```

Finally these values are passed through to the mongoDB chart we downloaded earlier.

### 6. Deploy your Helm Chart

First we need to build our docker images to deploy, from the root directory of the project run:

```sh
$ docker build -f frontend/Dockerfile -t frontend:v1.0.0 frontend
$ docker build -f backend/Dockerfile -t backend:v1.0.0 backend
```
or if you are using microk8s

```sh
$ docker build -f frontend/Dockerfile -t localhost:32000/frontend:v1.0.0 frontend
$ docker build -f backend/Dockerfile -t localhost:32000/backend:v1.0.0 backend

$ docker push localhost:32000/backend:v1.0.0
$ docker push localhost:32000/frontend:v1.0.0
```

Once the images are built you can now deploy your helm chart

```sh
$ helm install myapp chart/myapp
```

or if you are using microk8s 

```sh
$ microk8s helm3 install myapp chart/myapp
```

Check your pods are running by running:

```sh
$ kubectl get pods
```
or if you are using microk8s

```sh
$ microk8s kubectl get pods
```

You should get a similar output to:

```sh
NAME                                   READY   STATUS    RESTARTS   AGE
backend-deployment-5d6bb8c5f8-xm4bv    1/1     Running   0          67s
frontend-deployment-6c7779ff9d-xjdrv   1/1     Running   0          67s
myapp-mongodb-64df664c5b-st8fb         1/1     Running   0          66s
```

Also, by viewing the logs of the backend service with below command

```sh
kubectl logs backend-deployment-xxxxxxxx-xxxxx -f
```

or for microk8s

```sh
microk8s kubectl logs backend-deployment-xxxxxxxx-xxxxx -f 
```

you should get below message

`Connection is established with mongodb, details: mongodb://myapp-mongodb:27017`


No you can access your application at `http://localhost:30444/`

### Congratulations! 🎉

You have now created a Helm Chart which deploys a frontend and a backend whilst also calling and installing a dependency chart from the internet.

### Interacting with the application

By visiting `http://localhost:30444/` you should be able to see the UI as shown on the image below

![Application ui](/helm/images/frontend-initial-ui.png)

Lets add an item by filling the textbox and clicking on the `add new toDo` button

![Adding item](/helm/images/frontend-add-item.png)

After clicking the `Add new ToDo` button, the item should be added on the list.
You should also be able to see below message on the backend service logs.

`Creating Todo for NodeConfEU Clean the bicycle`

![Added item](/helm/images/frontend-added-item.png)

Click the x button next to the todo item and it should be removed

### Uninstalling the App

Once you are finished you can uninstall the chart by running:

```sh
$ helm uninstall myapp
```

or with microk8s

```sh
$ microk8s helm3 uninstall myapp
```