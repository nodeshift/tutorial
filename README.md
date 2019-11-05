# Building a Cloud Ready Express.js Application

This workshop shows you how to take a Node.js application and make it "cloud ready": adding support for Cloud Native Computing Foundation (CNCF) technologies using the package and templates provided by the CloudNativeJS project.

In this self-paced tutorial, you will:

* Create an Express.js application
* Add Health Checks and Metrics to your application
* Build your application with Docker
* Package your application with Helm
* Deploy your application to Kubernetes
* Monitor your application using Prometheus

The application you'll use is a simple Express.js app built using the Express Generator.

You'll learn about Health Checks, Metrics, Docker, Kubernetes, Prometheus and Grafana.

At the end you'll have a fully functioning application running as a cluster in Kubernetes, with production monitoring.

## Prerequisites

Before getting started, make sure you have the following prerequisites installed on your system.

1. [Node.js 8 or later](https://nodejs.org/en/download/)
2. Your IDE of choice
3. [Docker for Desktop](https://www.docker.com/products/docker-desktop)

## Setting up

### Enabling Kubernetes in Docker for Desktop

Ensure you have installed Docker for Desktop on your Mac and enabled Kubernetes within the app. To do so:

1. Select the Docker icon in the Menu Bar
2. Click Preferences > Kubernetes Tab > Enable Kubernetes.

It will take a few moments to install and start up, but when the indicator light in the bottom right corner is green, you're ready to go!

If you already use Kubernetes, ensure that you are configured to use the `docker-for-desktop` cluster. To do so:

1. Select the Docker icon in the Menu Bar
2. Click Kubernetes and select the `docker-for-desktop` context

### Installing Helm

Helm is a package manager for Kubernetes. By installing a Helm "chart" into your Kubernetes cluster you can quickly run all kinds of different applications. You can install Helm using one of the options below:

**Using a Package Manager:**

* macOS with Homebrew: `brew install kubernetes-helm`
* Linux with Snap: `sudo snap install helm`
* Windows with Chocolatey: `choco install kubernetes-helm`

**Using a Script:**

```sh
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

Once you have Helm ready, you can initialize the local CLI and also install Tiller into your configured Kubernetes cluster in one step:

```sh
helm init
```

This will install Tiller into the Kubernetes cluster, which controls the deployment of your Helm charts.

Now that Helm is installed, you should also configure access to the "stable" Helm repository as follows:

```sh
helm repo add stable https://kubernetes-charts.storage.googleapis.com
```
This makes it easy for you to install a number of applications and services into your Kubernetes cluster. You'll use this to install Prometheus and Grafana later in the workshop.

## 1. Create your Express.js Application

The `express-generator` package enables you to quickly create an application skeleton. The `express-generator` package installs the `express` command-line tool which you'll use to build you app.

Use the following steps to create your Express.js application:

1. Create a directory to host your project

   ```sh
   mkdir nodeserver
   cd nodeserver
   ```
2. Run the application generator with the npx command (available in Node.js 8.2.0).

   `npx express-generator`


For earlier Node versions, globally install and run the Express generator to build a skeleton application

   ```sh
   npm install --global express-generator
   express
   ```
   
This has built a simple Express.js application called `nodeserver`, after the name of the directory you are in.

Next install your applications dependencies and start your application:

1. Install  and start your applications NPM dependencies:
   
   ```sh
   npm install
   npm start
   ```

Your application should now be visible at the following URL: [http://localhost:3000](http://localhost:3000)

## 2. Add Health Checks to your Application

Kubernetes, and a number of other cloud deployment technologies, provide "Health Checking" as a system that allows the cloud deployment technology to monitor the deployed application and to take action should the application fail or report itself as "un-healthy".

The simplest form of Health Check is process level health checking, where Kubernetes checks to see if the application process still exists and restarts the container (and therefore the application process) if it is not. This provides a basic restart capability but does not handle scenarios where the application exists but is un-responsive, or where it would be desirable to restart the application for other reasons (such as low memory availability due to a memory leak).

The next level of Health Check is HTTP based, where the application exposes a "livenessProbe" URL endpoint that Kubernetes can make requests of in order to determine whether the application is running and responsive. Additionally, the request can be used to drive self-checking capabilities in the application.

The `@cloudnative/health-connect` package provides a Connect Middleware that makes it easy to add a default health check endpoint and provides a Promise based API for adding self-checking capabilities.

Add a Health Check endpoint to your Express.js application using the following steps:

1. Add the `@cloudnative/health-connect` dependency to your project
   
   ```sh
   npm install @cloudnative/health-connect
   ```

2. 	Set up a HealthChecker in `app.js`:

   ```js
   const health = require('@cloudnative/health-connect');
   let healthcheck = new health.HealthChecker();
   ```
   
3. Register a Liveness endpoint in app.js:

   ```js
   app.use('/health', health.LivenessEndpoint(healthcheck))
   ```
   
This has added a '/health' endpoint to your application. As no liveness checks are registered, this will return as status code of 200 OK and a JSON payload of `{"status":"UP","checks":[]}`.

Check that your livenessProbe Health Check endpoint is running:

1. Start your application:

   ```sh
   npm start
   ```
2. Visit the `health` endpoint:

   [http://localhost:3000/health](http://localhost:3000/health)

For information on how to register health/liveness checks, and additional support for start-up, readiness and shutdown checks, see the [Cloud Health documentation](https://github.com/CloudNativeJS/cloud-health/blob/master/README.md).


## 3. Add Metrics to your Application


For any application deployed to a cloud, it is important that the application is "observable": that you have sufficient information about an application and its dependencies such that it is possible to discover, understand and diagnose the state of the application. One important aspect of application observability is metrics-based monitoring data for the application. 

The CNCF recommended metrics system is [Prometheus](http://prometheus.io), which works by collecting metrics data by making requests of a URL endpoint provided by the application. Prometheus is widely supported inside Kubernetes, meaning that Prometheus also collects data from Kubernetes itself, and application data provided to Prometheus can also be used to automatically scale your application. 

The `appmetrics-prometheus` package provides an easy to use library that auto-instruments your application to collect metrics and exposes them on a `/metrics` endpoint for consumption by Prometheus.

Add a `/metrics` Prometheus endpoint to your Express.js application using the following steps:

1. Add the `appmetrics-prometheus` dependency to your project
   
   ```sh
   npm install appmetrics-prometheus
   ```

2. 	Require `appmetrics-prometheus` and attach it to your Express server:

   ```js
   var prometheus = require('appmetrics-prometheus').attach();
   ```
   
This has added a '/metrics' endpoint to your application. This automatically starts collecting CPU, Memory and HTTP responsiveness data from your application and exposes it in a format that Prometheus understands.

Check that your metrics endpoint is running:

1. Start your application:

   ```sh
   npm start
   ```
2. Visit the `metrics` endpoint:

   [http://localhost:3000/metrics](http://localhost:3000/metrics)

For information on how to configure the appmetrics-prometheus library see the [appmetrics-prometheus documentation](https://github.com/CloudNativeJS/appmetrics-prometheus).

You can install a local Prometheus server to graph and visualize the data, and additionally to set up alerts. For this workshop you'll use Prometheus once you've deployed your application to Kubernetes.

## 4. Building your Application with Docker

Before you can deploy your application to Kubernetes, you first need to build your application into a Docker container and produce a Docker image. This packages your application along with all of its dependencies in a ready to run format.

CloudNativeJS provide a "[Docker](https://github.com/CloudNativeJS/docker)" project that provides a number of best-practice Dockerfile templates that can be used to build your Docker container and produce your image.

For this workshop, you'll use the `Dockerfile-run` template, which builds a production-ready Docker image for your application.

Build a production Docker image for your Express.js application using the following steps:

1. Copy the `Dockerfile-run` template into the root of your project:

   ```sh
   wget https://raw.githubusercontent.com/CloudNativeJS/docker/master/Dockerfile-run
   ```
   
1. Copy the `.dockerignore` file into the root of your project:

   ```sh
   wget https://raw.githubusercontent.com/CloudNativeJS/docker/master/.dockerignore
   ```
   
2. Build the Docker run image for your application:

   ```sh
   docker build -t nodeserver-run:1.0.0 -f Dockerfile-run .
   ```
   
You have now built a Docker image for your application called `nodeserver-run` with a version of `1.0.0`. Use the following to run your application inside the Docker container:

```sh
docker run -i -p 3000:3000 -t nodeserver-run:1.0.0
```

This runs your Docker image in a Docker container, mapping port 3000 from the container to port 3000 on your laptop so that you can access the application. 

Go ahead and visit your applications endpoints to check that it is running successfully: 

* Homepage: [http://localhost:3000/](http://localhost:3000/)
* Health: [http://localhost:3000/health](http://localhost:3000/health)
* Metrics: [http://localhost:3000/metrics](http://localhost:3000/metrics)

For information on Docker usage, how to develop your application directly in a Docker container, and how to debug your application in a Docker container, see the [Docker documentation](https://github.com/CloudNativeJS/docker).

## 5. Packaging your Application with Helm

In order to deploy your Docker image to Kubernetes you need to supply Kubernetes with configuration on how you need your application to be run, including which Docker image to use, how many replicas (instances) to deploy and much memory and CPU to provide to each.

Helm charts provide an easy way to package your application with this information. 

CloudNativeJS provides a "[Helm](https://github.com/CloudNativeJS/helm)" project that provides a template best-practice Helm chart template that can be used to package your application for Kubernetes.

Add a Helm chart for your Express.js application using the following steps:

1. Download the template Helm chart

   ```sh
   wget https://github.com/CloudNativeJS/helm/archive/master.zip
   ```

2. Unzip the downloaded template chart

   ```sh
   unzip master.zip 
   ```

3. Move the chart to your projects root directory

   ```sh
   mv helm-master/chart chart
   rm -rf helm-master master.zip
	```
	
The provided Helm chart provides a number of configuration files, with the configurable values extracted into `chart/nodeserver/values.yaml`. In this file you provide the name of the Docker image to use, the number of replacates (instances) to deploy, etc.

Go ahead and modify the `chart/nodeserver/values.yaml` file to use your image, and to deploy 3 replicas:

1. Open the `chart/nodeserver/values.yaml` file
2. Change the `repository` field to `nodeserver-run`
3. Change the `pullPolicy` to `IfNotPresent`
4. Change the `replicaCount` value to `3`

The `repository` field gives the name of the Docker image to use. The `pullPolicy` change tells Kuberentes to use a local Docker image if there is one available rather than always pulling the Docker image from a remote repository. Finally, the `replicaCount` states how many instances to deploy.
 

## 6. Deploying your Application to Kubernetes

Now that you have built a Helm chart for your application, the process for deploying your application has been greatly simplified.

Deploy your Express.js application into Kubernetes using the following steps:

1. Deploy your application into Kubernetes:

   ```sh
   helm install --name nodeserver chart/nodeserver
   ```
   
2. Ensure that all the "pods" associated with your application are running:

   ```sh
   kubectl get pods
   ```
   
Now everything is up and running in Kubernetes. Its not possible to navigate to localhost:3000 as usual because your cluster isn't part of the localhost network, and because there are several instances to choose from.

You can forward one of the instances ports to your laptop using the following steps:

1. Choose one of the running "pod" instances:
   ```sh
   kubectl get pods #Copy the nodeserver NAME
   ```
   
2. Forward its port to your laptop

   ```sh
   kubectl port-forward nodeserver-deployment-XXXXXX-XXXXX 3000:3000
   ```

You can now access that pod's endpoints from your browser.

## 7. Monitoring your Application with Prometheus

Installing Prometheus into Kubernetes can be done using its provided Helm chart:

```sh
helm install stable/prometheus --name prometheus --namespace prometheus
```

You can then run the following two commands in order to be able to connect to Prometheus from your browser:

```sh
export POD_NAME=$(kubectl get pods --namespace prometheus -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace prometheus port-forward $POD_NAME 9090
```
You can now connect to Prometheus at the following address:

* [http://localhost:9090](http://localhost:9090)

This should show the following screen:
![prometheus-dashboard](./resources/prometheus-dashboard.png)
Prometheus will be automatically collecting data from your Express.js application, allowing you to create graphs of your data.

To build your first graph, type `os_cpu_used_ratio` into the **Expression** box and click on the **Graph** tab:

![prometheus-graph](./resources/prometheus-graph.png)


Whilst Prometheus provides the ability to build simple graphs and alerts, Grafana is commonly used to build more sophisticated dashboards.

### Installing Grafana into Kubernetes

Installing Grafana into Kubernetes can be done using its provided Helm chart:

```sh
helm install stable/grafana --set adminPassword=PASSWORD --name grafana --namespace grafana
```

You can then run the following two commands in order to be able to connect to Grafana from your browser:

```sh
export POD_NAME=$(kubectl get pods --namespace grafana -l "app=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace grafana port-forward $POD_NAME 3000
```
You can now connect to Grafana at the following address, using `admin` and `PASSWORD` to login:

* [http://localhost:3000](http://localhost:3000)

This should show the following screen:

![grafana-home](./resources/grafana-home.png)

In order to connect Grafana to the Prometheus service, next click on **Add data source**.

This opens the a panel that should be filled out with the following entries:

* Name: `Prometheus`
* Type: `Prometheus`
* URL: `http://prometheus-server.prometheus.svc.cluster.local`

![grafana-datasource](./resources/grafana-datasource.png)

Now click on **Save & Test** to check the connection and save the Data Source configuration.

Grafana now has access to the data from Prometheus.

### Installing a Kubernetes Dashboard into Grafana

The Grafana community provides a large number of pre-created dashboards which are available for download, including some which are designed to display Kubernetes data.

To install one of those dashboards, click on the **+** icon and select **Import**

![grafana-import-select](./resources/grafana-import-select.png)

In the provided panel, enter `1621` into the **Grafana.com Dashboard** field in order to import dashboard number 1621, and press **Tab**.

This then loads the information on dashboard `1621` from Grafana.com.

Set the **Prometheus** field to `Prometheus` and click **Import**.

![grafana-dashboard-import](./resources/grafana-dashboard-import.png)

This will then open the dashboard, which will automatically start populating with data about your Kubernetes cluster.

![grafana-kube-dash](./resources/grafana-kube-dash.png)

### Adding Custom Graphs

In order to extend the dashboard with your own graphs, click the **Add panel** icon on the top toolbar and select **Graph**.

![grafana-add-graph](./resources/grafana-add-graph.png)

This creates a blank graph. Select the **Panel Title** pull down menu and select **Edit**.

This opens an editor panel where you can select data that you'd like to graph.

Type `os_cpu_used_ratio` into the data box, and a graph of your applications CPU data will show on the panel.

You can create more complex queries and apply filters according to any kubernetes value. For example, the following will show all of the HTTP request durations for your specific application:

* `http_request_duration_microseconds{kubernetes_name="nodeserver-service"}`


You now have integrated monitoring for both your Kubernetes cluster and your deployed Express.js application.

Here are some ideas you could explore to further your learning.

* Add a Singlestat that shows how many instances of you Express.js application are currently running
* Add a Singlestat that shows how many requests your Express.js app has responded to


## Congratulations! 🎉

You now have an Express.js application deployed at scaling using Docker and Kubernetes, with automatic restart and full metrics based monitoring enabled!
