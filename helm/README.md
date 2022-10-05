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

1. Install [Node.js 16](https://nodejs.org/en/download/) (or use [nvm](https://github.com/nvm-sh/nvm#installing-and-updating) for linux, mac or [nvm-windows](https://github.com/coreybutler/nvm-windows#installation--upgrades) for windows)
1. Podman v4 (and above)
   - **On Mac**: [Podman](https://podman.io/getting-started/installation#macos)
   - **On Windows**: Skip this step, as for installing Podman you will get prompt during Podman Desktop installation.
   - **On Linux**: [Podman](https://podman.io/getting-started/installation#installing-on-linux)
1. Podman Desktop
   - **On Mac**: [Podman Desktop](https://podman-desktop.io/downloads/macOS)
   - **On Windows**: [Podman Desktop](https://podman-desktop.io/downloads/windows)
   - **On Linux**: [Podman Desktop](https://podman-desktop.io/downloads/linux)
1. Kubernetes
   - **On Mac**: [minikube](https://minikube.sigs.k8s.io/docs/start/)
   - **On Windows**: [minikube](https://minikube.sigs.k8s.io/docs/start/)
   - **On Linux**: [minikube](https://minikube.sigs.k8s.io/docs/start/)
1. Helm v3 - [Installation](./README.md#installing-helm-v37)
   - **Note**: This workshop tested with Helm v3.7

## Setting up

### Starting Podman Machine

#### Linux

Nothing to do, no Podman machine is required on Linux

#### On Mac

After installing podman, open a terminal and run below commands to initialize and run podman machine:

_**NOTE:** \*On Apple M1 Pro chip, system version has to be 12.4 and above._

```
podman machine init --cpus 2 --memory 8096 --disk-size 20
podman machine start
podman system connection default podman-machine-default-root
```

#### On Windows

1. Launch Podman Desktop and on the home tab click on **install podman**. In case of any missing parts for podman installation (e.x. wsl, hyper-v, etc.) follow the instructions indicated by Podman Desktop on the home page. In that case you might need to reboot your system several times.

1. After installing podman, set WSL2 as your default WSL by entering below command in PowerShell (with administration priviledges).

   ```
   wsl --set-default-version 2
   ```

1. On Podman Desktop Home tab -> click on **initialize Podman** -> wait till the initialization is finished
1. On Podman Desktop Home tab -> click on **Run Podman** to run podman.

#### On Windows **Home**

1. Downlodad podman from https://github.com/containers/podman/releases the Windows installer file is named podman-v.#.#.#.msi
1. Run the MSI file
1. Launch as Administrator a new Command Prompt
1. On the Command Prompt run:
   ```
   podman machine init
   podman machine set --rootful
   podman machine start
   ```
1. Launch Podman Desktop to see and manage your containers, images, etc.

### Starting Kubernetes

#### On Mac:

1. Install Minikube

   <details>
      <summary>Download binary file (click to expand)</summary>

   **x86-64**

   ```
   curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
   ```

   **ARM64**

   ```
   curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-arm64
   ```

   </details>

   Add minikube binary file to your `PATH system variable`

   ```
   chmod +x minikube-darwin-*
   mv minkube-linux-* /usr/local/bin/minikube
   ```

1. start minikube
   ```
   minikube start --driver=podman --container-runtime=cri-o
   ```

#### On Windows:

1. Download minikube
   ```
   https://github.com/kubernetes/minikube/releases/latest/download/minikube-windows-amd64.exe
   ```
1. Rename `minikube-windows-amd64.exe` to `minikube.exe`
1. Move minikube under `C:\Program Files\minikube` directory

1. Add `minikube.exe` binary to your `PATH system variable`

   1. Right-click on the **Start Button** -> Select **System** from the context menu -> click on **Advanced system settings**
   1. Go to the **Advanced** tab -> click on **Environment Variables** -> click the variable called **Path** -> **Edit**
   1. Click **New** -> Enter the path to the folder containing the binary e.x. `C:\Program Files\minikube` -> click **OK** to save the changes to your variables
   1. Start Podman Desktop and click on run podman

1. Start minikube:

   - For windows Start minikube by opening Powershell or Command Prompt **as administrator** and enter below command.

   ```
   minikube start
   ```

   - For windows **Home** Start minikube by opening Powershell or Command Prompt **as administrator** and enter below command.

   ```
   minikube start --driver=podman --container-runtime=containerd
   ```

#### On Linux

1. Install Minikube

   <details>
      <summary>Download binary file (click to expand)</summary>

   **x86-64**

   ```
   curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
   ```

   **ARM64**

   ```
   curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-arm64
   ```

   **ARMv7**

   ```
   curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-arm
   ```

   **ppc64**

   ```
   curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-ppc64le
   ```

   **S390x**

   ```
   curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-s390x
   ```

   </details>

   Add minikube binary file to your `PATH system variable`

   ```
   chmod +x minikube-linux-*
   mv minkube-linux-* /usr/local/bin/minikube
   ```

1. Change minikube for starting podman rootless
   https://minikube.sigs.k8s.io/docs/drivers/podman/#rootless-podman

   ```
   minikube config set rootless true
   ```

1. start minikube

   ```
   minikube start  --driver=podman --container-runtime=containerd
   ```

1. Possible additional steps needed
   - delegation also needed on Unbuntu 2022 - https://rootlesscontaine.rs/getting-started/common/cgroup2/

### Installing Helm v3.7

Helm is a package manager for Kubernetes. By installing a Helm "chart" into your Kubernetes cluster you can quickly run all kinds of different applications. You can install Helm by downloading the binary file and adding it to your PATH:

1. Download the binary file from the section **Installation and Upgrading** for Operating system accordingly.

   - https://github.com/helm/helm/releases/tag/v3.7.2

1. Extract it:

   - On Linux:
     ```
     tar -zxvf helm-v3.7.2-*
     ```
   - On Windows: **Right Click** on `helm-v3.7.2-windows-amd64` zipped file -> **Extract All** -> **Extract**
   - On Mac:
     ```
     tar -zxvf helm-v3.7.2-*
     ```

1. Add helm binary file to your `PATH system variable`

   On Linux and Mac (sudo required for cp step on linux):

   ```
   cp `./<your-linux-distro>/helm` /usr/local/bin/helm
   rm -rf ./<your-linux-distro>
   ```
   
   If running on Mac results in a pop up indicating that the app could not be verified,
   you will need to go to Apple menu > System Preferences, click Security & Privacy and
   allow helm to run.
   
   On Windows:

   1. Move helm binary file to `C:\Program Files\helm`
   1. Right-click on the **Start Button** -> Select **System** from the context menu -> click on **Advanced system settings**
   1. Go to the **Advanced** tab -> click on **Environment Variables** -> click the variable called **Path** -> **Edit**
   1. Click **New** -> Enter the path to the folder containing the binary e.x. `C:\Program Files\helm` -> click **OK** to save the changes to your variables

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
   cd myapp
   helm dependency update
   cd ..
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

Check your pods are running by running:

```sh
$ kubectl get pods
```

You should get a similar output to:

```sh
NAME                                   READY   STATUS    RESTARTS   AGE
backend-deployment-5d6bb8c5f8-xm4bv    1/1     Running   0          67s
frontend-deployment-6c7779ff9d-xjdrv   1/1     Running   0          67s
myapp-mongodb-64df664c5b-st8fb         1/1     Running   0          66s
```

Wait for some time until mongo status is running

Also, by viewing the logs of the backend service with below command

```sh
kubectl logs backend-deployment-xxxxxxxx-xxxxx -f
```
you should get below message

`Connection is established with mongodb, details: mongodb://myapp-mongodb:27017`


No you can access your application at `http://localhost:30444/`

### Congratulations! 🎉

You have now created a Helm Chart which deploys a frontend and a backend whilst also calling and installing a dependency chart from the internet.

### Interacting with the application

By visiting `http://localhost:30444/` you should be able to see the UI as shown on the image below

![Application ui](/helm/images/frontend-initial-ui.png)

Lets add an item by filling the textbox and clicking on the `Add new toDo` button

![Adding item](/helm/images/frontend-add-item.png)

By clicking the `Add new ToDo` button, the item should be added on the list as shown on image below.

You should also be able to see below message on the backend service

`Creating Todo for NodeConfEU Clean the bicycle`

by viewing the logs with below command 
```sh
kubectl logs backend-deployment-xxxxxxxx-xxxxx -f
```

![Added item](/helm/images/frontend-added-item.png)

Click the x button next to the todo item and it should be removed

### Uninstalling the App

Once you are finished you can uninstall the chart by running:

```sh
$ helm uninstall myapp
```