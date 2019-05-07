# Deploying the BlueCompute Microservices Application

These exercises deploy portions of the BlueCompute application. This microservices application is created as a showcase of developing IBM Cloud capabilities. You will use Jenkins to deploy the application components. The result is **not** a fully working application as this is just a demonstration of how to use Jenkins to build and deploy microservices.

## Exercise 1: Designing the  Microservices Deployment

The application is shown in the following diagram. <br>

![Application architecture](images/blue/arch.jpg)

1. The BlueCompute application contains a frontend layer, a backend layer (microservices) and a data layer (backing services).

2. Let's go through this exercise to design how you will group and deploy the application. The deployment should start with the data layer, followed by the backend layer and the frontend layer.

3. For the following data services resource, which Kubernetes resources should you define?

	- ElasticSearch: _______________________________
	- CouchDB: _______________________________
	- Orders MySQL: _______________________________
	- Inventory MySQL: _______________________________

4. For the following backend services microservice applications, which Kubernetes resources should you define?

	- Catalog microservice: _______________________________
	- Customer microservice: _______________________________
	- Orders microservice: _______________________________
	- Inventory microservice: _______________________________

5. For the following frontend services application, which Kubernetes resources should you define? Assume for now that the Ingress resource will be deployed with the Web application and that you will ignore the swift Web BFF.

	- Authentication: _______________________________
	- Web BFF: _______________________________

6. Now that you have been through the design part, let's consider the following:

	- What resource(s) is/are common for all components?
<br>_______________________________
	- What resource(s) is/are common for the data layer?
<br>_______________________________
	- When would you have an ingress resource?
<br>_______________________________

## Exercise 2: Setting Up and Verifying Backing Services

The following instructions allow you to build and deploy one of the backend services required by the BlueCompute application.

**Note**: Better than just following these steps, you can learn much more by looking at the Jenkinsfile and helm charts you will be working with. This will help you understand what actually happens in the process.

1. Use a terminal session on the **Boot VM** to define a secret to access the IBM Cloud Private user:

	- Encode the user and password that is used to connect to ICP in base64:

		``echo admin | base64``

	- Create a ICP_secret.yaml file in your current directory with the following contents:

    ```
		apiVersion: v1
		kind: Secret
		metadata:
		  name: icpadmin
		type: Opaque
		data:
		  username: YWRtaW4K
		  password: YWRtaW4K
    ```

	- Load the secret to ICP

		``kubectl create -f ICP_secret.yaml -n jenkins``

2. Set up IBM Cloud Private registry parameters as a ConfigMap (namespace, imagePullSecret and registry).

	- Create a ICP_config.yaml file in your current directory with the following contents:

    ```
		apiVersion: v1
		kind: ConfigMap
		metadata:
		  name: icpconfig
		data:
		  namespace: default
		  registry: mycluster.icp:8500
    ```

	- Load the ConfigMap to ICP

		``kubectl create -f ICP_config.yaml -n jenkins``

3. In your browser, go to the Jenkins Web UI at `http://10.10.1.4/jenkins`. Log in as `admin` with a password of `admin_0000`. You should be in the Jenkins dashboard.

4. Click **New item** from the Jenkins menu on the left toolbar. <br>

5. Enter a name of `CouchDB`, select **Pipeline** and click **OK**.<br>

   ![Create pipeline](images/blue/couchdb1.jpg)

6. Scroll down to the Pipeline section and specify:

	- Definition: `Pipeline script from SCM`
	- SCM: `Git`
	- Repository URL: `https://github.com/davemulley/icp-jenkins-helm-bluecompute`
	- Branch Specifier: `*/master`
	- Script Path: `charts/ibmcase-couchdb/Jenkinsfile `
	- Click **Save**

	 ![Pipeline from git](images/blue/couchdb2.jpg)

7. On the couchDB pipeline page, click **Build now**.<br>

8. Once the pipeline is running as indicated on the lower left side, open the drop down menu next to the run number and select **Console Output**. <br>

   ![Running pipeline](images/blue/couchdb3.jpg)

9. The console should show the helm chart being deployed at the end and the pipeline having finished successfully.<br>

   ![Console output](images/blue/couchdb4.jpg)

10. Having the helm chart deployed does not necessarily mean that the application is correctly deployed. You must check whether the actual application pod is running. This may take a couple of minutes depending on the network speed to load the container. Run `kubectl get pod` commands and wait until the pod for couchdb is running. <br>

   ![Running pod](images/blue/couchdb5.jpg)

11. Check the status of the `bluecompute-couchdb` release, by running `helm status bluecompute-couchdb --tls`. <br>

   ![CouchDB status](images/blue/couchdb6.jpg)<br>

	 Can you identify the chart's components?

	- Secret: `binding-customer-couchdb`
<br>___________________________________________
	- Service: `bluecompute-couchdb-couchdb`
<br>___________________________________________
	- Deployment: `bluecompute-couchdb--couchdb`
<br>___________________________________________
	- Job: `bluecompute-couchdb-ibmcase-couchdb-create-user-"*"  `
<br>___________________________________________
	- Pods: `bluecompute-couchdb--couchdb-"*" `and `bluecompute-couchdb-ibmcase-couchdb-create-user-"*" `
<br>___________________________________________

   **Note:** The deployment of the other backend components (elasticsearch, inventory-mysql and orders-mysql) is actually very similar. You create the Jenkins pipelines from SCM (GIT) and run them individually. These pipelines are not automated (triggered using code changes) as this is meant to be a stable backend. In a real production environment, you would want to add a PersistentVolumeClaim to physically host the data instead of storing it in volatile containers as this example describes.

12. Open a browser and navigate to `https://github.com/davemulley/icp-jenkins-helm-bluecompute/tree/master/charts/ibmcase-couchdb`. Review the `Jenkinsfile`

 	Can you determine what the Jenkinsfile is doing?

## Exercise 3: Deploying Microservices Components

The set of charts in this exercise is different that in the previous set. The main difference is that the docker images are built by the pipeline and loaded to the IBM Cloud Private local registry.

**Note**:

These instructions load the Jenkinsfile directly from a forked repository that has been pre-configured. The actual steps should have been:

- Fork the repository.
- Create a Jenkinsfile.
- Poll the repository for commit changes and create a post-commit hook.

You would never need to build the pipeline manually.

1. There are 4 microservices built into the BlueCompute application: customer, inventory, catalog and orders. You will deploy one of them here for demonstrations purposes

2. Create a new Jenkins pipeline using the following specifications for the `customer` application.

	- Name: `Customer`
	- Repository URL: `https://github.com/davemulley/refarch-cloudnative-micro-customer `
	- Branch Specifier: `*/master`
	- Script Path: `Jenkinsfile`

	- Run the pipeline using the **Build now** link.

	- View the `console` output of the build job

	- Make sure the `customer` pod is running

3. Check the status of the `bluecompute-customer` release, by running `helm status bluecompute-customer --tls`. <br>

    ![orders-mysql status](images/blue/customer1.jpg)

4. Open a browser navigate to `https://github.com/davemulley/refarch-cloudnative-micro-customer` and review the `Jenkinsfile`

	How is this Jenkinsfile building the Docker Image?

	How is this Jenkinsfile deploying the application?

## Conclusion
In this exercise you have used Jenkins to perform tasks including deploying a helm chart, compiling code, building a Docker Image and also pushing a Docker Image to the ICP Registry. Jenkins dynamically created the `slaves` to perform the following steps which are the types of steps that you will automate for microservices as well as applications that have been modernized to run in IBM Cloud Private. 

#### *** End of exercises ***
