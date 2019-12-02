How To Use Kubernetes, and a Simple Example
Now that we have gone through the basics of Kubernetes, we will take a detailed look at a simple example. In this example, we will deploy a web application using Kubernetes. We will be using Docker as our container runtime. The application in our example has three distinct parts:
Database (MySQL server)
Back end (Java Spring Boot application)
Front end (Angular app)
We will deploy all of these components on a Kubernetes cluster. We will have one replica of database, two replicas of back end and two replicas of front end. Front-end instances will communicate with back end through HTTP. Back-end instances will communicate with the database. To facilitate this communication, we have to configure Kubernetes accordingly.
We will configure the cluster by creating Kubernetes objects. These Kubernetes objects will contain the desired state of our deployment. Once these objects are persisted into the cluster state store, the internal architecture of Kubernetes will take necessary steps to ensure that the abstract state in the cluster state store is the same as the physical state of the cluster.
We will use kubectl to create the objects. Kubernetes supports both imperative and declarative ways of creating objects. Production environments are generally configured by the declarative approach. We will use the declarative approach in this example. For each object, we will first prepare a manifest file, a yaml file containing all the information related to the object. Then we will execute the kubectl command, kubectl apply -f <FILE_NAME>to persist the object in the cluster state store.
We will first containerize the application code we have implemented. After this, we will configure the deployment of our database followed by back end. We will finish the example by configuring the front end.
Step 1. Containerize the application and upload image to container image registry
The first step would be to create a container image of the application we have implemented and upload it to container registry.
A container image is a packaged form of the containerized application. It can be transferred across computers, just like any normal file. The container runtime environment can create a running instance of a containerized application using the container image.
Container registry is generally the centralized repository where container images are stored. One could upload container images to a container registry and download them wherever and whenever they are needed. There are numerous container registry services available: Azure Container Registry, Google Container Registry, Amazon ECR, etc. We will use Docker hub for this example, but one could use any image registry (public or private) that fits their use case.
The application we are going to deploy has front end implemented with Angular Framework, and back end implemented with Spring Boot Framework. Links to GitHub repositories containing the code are provided in the final section of this piece. Once we have implemented the code as per our requirements, we will build executables with build tools (Angular CLI and Maven in this case).
Now we will create container images by building docker images using Dockerfile . Dockerfile for both front end and back end used in this example are shown below.

Dockerfile of Front-end angular application

Dockerfile of Back-end spring boot application
Once container images are created, we can upload these to any container image registry. Here we will upload these images to Docker Hub. We have uploaded the front-end image with the namekubernetesdemo/to-do-app-frontend, and the back-end image with the name kubernetesdemo/to-do-app-backend. We will obtain the database image from the official MySQL docker repository mysql. Official Docker images generally do not have any prefix, like mysql. Unofficial images are required to have a prefix like kubernetesdemo/here.
We have to mention the name of these images in the Kubernetes manifest files, which we will see below. Kubernetes will fetch and run these images on respective cluster nodes whenever required.
Step 2. Set up Kubernetes cluster and CLI
There are numerous solutions available for setting up a Kubernetes cluster. Different Kubernetes solutions meet different requirements: ease of maintenance, security, control, available resources, and expertise required to operate and manage a cluster. One could refer to the official documentation for more details about how a cluster could be set up. This example has been replicated on both local (Minikube) and cloud provider (GKE) setup. Kops is a project that aims to simplify the Kubernetes cluster setup process.
As mentioned earlier, we will use kubectl as our CLI. Instructions for installing kubectl can be found here. Once kubectl is installed, it should be configured to communicate with the Kubernetes cluster we have set up. In the case of Minikube, minikube start command will automatically configure kubectl. For cloud setup, instructions can be found in their respective quick start guide(eg: GKE).
Step 3. Database configuration setup
Back-end instances need to communicate with the database. All the configuration details required to connect with the database are stored in a configuration file.
Let’s take a look at the back-end spring configuration file in this example

Back-end Configuration File
This configuration file expects some environment variables, like DB_USERNAME, DB_PASSWORD, DB_HOST, DB_NAME . We will pass the values of these variables to Kubernetes through configMaps and secrets. Then we will configure the back-end pod to read the environment variable from the configMaps and secrets .
The MySQL Database docker image expects some environment variables. We will need to configure the following environment variables MYSQL_ROOT_PASSWORD, MYSQL_USER, MYSQL_PASSWORD, MYSQL_DATABASE.
Now that we have an idea about the configuration required for our application, we will create configMaps and secrets in our Kubernetes cluster with required data.
First, to hold database specific information, we will create one configMap and two secrets. The configMap will contain non-sensitive information about the database setup, like the location where the database is hosted and the name of the database. We will define a Kubernetes service, this will expose the location of the database. Kuberebetes DNS will resolve the service name to actual ip address of the database during runtime. Below is the configMap, which is used to store non-sensitive information related to the database in this example

We will use two secrets to store sensitive data. The first secret will contain the database root user credentials and the second secret will contain application user credentials. Below are these two files

Database Root User Credentials

Database application user Credentials
By executing kubectl apply -f <FILE_NAME>we will create the ConfigMap and Secret objects in our Kubernetes cluster state store. We stored the values of host and name in this ConfigMap object and username and password in Secret object. We will access thesesecrets and ConfigMaps in later steps to configure our deployments .
Similarly, the configuration of front end expects environment variable SERVER_URI which will indicate where back end is hosted. We will create this configMap after configuring back-end Deployment
Step 4. Configure PVC, service, and deployment for database
Our next step would be to create the services and deployments required for our database setup. Below is the file which creates relevant Kubernetes Service and Kubernetes Deployment for the database setup in this application.

Configuration for Deploying MySQL Database on Kubernetes Cluster
Through this file, we created multiple Kubernetes objects. First, we created a Kubernetes Service with the name mysql for accessing the pod running the MySQL container. Next, we created a Persistent Volume Claim (PVC) of one GB, this will result in Kubernetes cluster dynamically allocating the required persistent storage for MySQL (enable default dynamic storage, if it is not enabled in your cluster). After this, we created a Deployment object, which configures the deployment of MySQL Server in the cluster. Into the MySQL container, we injected environment variables like MYSQL_ROOT_PASSWORD, MYSQL_USER, MYSQL_PASSWORD , and MYSQL_DATABASE using the configMaps and services we created in the previous step.
Step 5. Configure service and deployment for back end
Next, we set up our back-end application deployment. Below is the yaml file which creates the required Kubernetes objects.

Here we first created a Service of type LoadBalancer (use NodePort if you are running Kubernetes locally) which exposes the back-end instances. Loadbalancer type provides an External-IP , through which one could access the back-end services externally. (use minikube ipwith the port if you are using minikube). Next, we created the Deployment object configured to contain two replicas of the back-end instance. And then injected the required environment variables from the configMaps and secrets we have created earlier. This deployment will use the image kubernetesdemo/to-do-app-backend which we created in step one.
Step 6. Front-end configuration setup
Front end expects the value of External-IP of back end, generated in the above step, to be passed in the form of the environment variable SERVER_URI. We will now create a config map to store this information related to the back-end setup.

ConfigMap storing information related to back-end server URI
We will use this configMap to inject SERVER_URI value when configuring the deployment of front end, in the next step.
Step 7. Configure service and deployment for front end
Next, we set up our front-end application deployment. Below is the yaml file which creates the required Kubernetes objects.

Here, we first created a Service of type LoadBalancer (use NodePort if you are running Kubernetes locally) which exposes the front-end instances. Loadbalancer type provides an External-IP , through which one could access the front-end services externally. (use minikube ipwith the port if you are using minikube). Next, we created the Deployment object configured to contain two replicas of the front-end instance. This deployment will use the image kubernetesdemo/to-do-app-frontend, which we created in step one. After this, we injected the environment variable SERVER_URI from configMap, which we created in the above setup.
That’s it. Our simple application is now completely deployed. After this, the front end of the application should be accessible using front-end service External-IP from any browser. The Angular app will call the back end through HTTP and back end will communicate with MySQL database, where our application data is persisted. The image below shows the overall setup described in this example

High-Level View of To-Do-App Deployment Setup on Kubernetes
This entire deployment is now managed by Kubernetes. If one of the pods goes down for unknown reasons, Kubernetes will bring up a new pod without any manual intervention. Using kubectl, we could monitor and update this deployment, whenever required.
