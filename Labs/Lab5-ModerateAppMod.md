# Modernize Moderate App (come code changes required)

The goal of this part of mock engagement is to modernize the Customer Order (a.k.a. "Purple Compute") application running on traditional WAS to run on IBM Cloud Private.

Customer Order is a MODERATE complexity app for modernization purposes, as source code changes required (even thought very minimal).
Customer Order uses DB2 backend and LDAP for user authentication.

## Task 1. Setup pre-loaded DB2 image with pre-configured application database

#### Create Pre-loaded DB2 instance

Create a lab folder
```bash
mkdir /root/lab5
cd /root/lab5
```

Download base DB2 container image:
```bash
docker pull ibmcom/db2express-c:latest
```

Start the DB2 container:
```bash
docker run -d -e LICENSE=accept -e DB2INST1_PASSWORD=passw0rd -p 50000:50000 --name purple-compute-db-preloaded ibmcom/db2express-c:latest db2start
```

Open a shell into DB2 container:
```bash
docker exec -it purple-compute-db-preloaded  bash
```

Switch to db2 user defined in the base image
```bash
su - db2inst1
```
(login using passw0rd)

Create application database (this may take a few seconds to complete):

```bash
 db2 create DB ORDERDB
 DB20000I  The CREATE DATABASE command completed successfully.
```

Populate the database. Use pre-built script to pull all the needed ddl and sql files from git repo
https://github.com/ibm-cloud-architecture/refarch-jee-customerorder/tree/liberty/Common

```bash
 su - ${DB2INSTANCE} -c "bash <(curl -s https://raw.githubusercontent.com/ibm-cloud-architecture/refarch-jee-customerorder/liberty/Common/bootstrapCurlDb2.sh)"
```

Observe progress ending with
```bash
Database 'ORDERDB' bootstrapped for application use.
```

You can now exit from the db2inst1 session and then from the container shell
```bash
exit
exit
```

Commit the image, as it now has the data we want on it.
```bash
 docker commit purple-compute-db-preloaded mycluster.icp:8500/default/purple-compute-db-preloaded:latest
```

Stop the docker container and delete it.
```bash
 docker stop purple-compute-db-preloaded
 docker rm purple-compute-db-preloaded
```

#### Push pre-loaded DB2 instance to ICP
Now we can push the pre-loaded DB2 image to ICP:
```bash
 docker push  mycluster.icp:8500/default/purple-compute-db-preloaded
```

#### Create and run pre-loaded DB2 container on ICP
Create a file named `deploy.yaml` in `/root/lab5` with the following contents:

```bash
apiVersion: v1
kind: Service
metadata:
  name: "purple-compute-db-preloaded"
  namespace: "default"
spec:
  type: NodePort
  ports:
  - name: db2
    port: 50000
    protocol: "TCP"
    targetPort: 50000
  selector:
    app: "purple-compute-db-preloaded"
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: "purple-compute-db-preloaded"
  namespace: "default"
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: "purple-compute-db-preloaded"
    spec:
      containers:
      - name: purple-compute-db-preloaded
        image: mycluster.icp:8500/default/purple-compute-db-preloaded
        args: ["db2start"]
        env:
        - name: LICENSE
          value: "accept"
        - name: DB2INST1_PASSWORD
          value: "passw0rd"
```

4. Run the following command to create the deployment and service

```bash
kubectl create -f deploy.yaml
```

5. Run the following command to get the NodePort that has been assigned to the service

```bash
kubectl get services -n default
```

![pbw](images/lab5/service.jpg)

In our case DB2 endpoint will be `10.10.1.4:32518`

The step by step modernization guide for Customer Order application is provided here (referred to later  as "the guide"):

https://www.ibm.com/cloud/garage/content/course/websphere-on-cloud-private

*IMPORTANT:* The instructions in this modernization guide are referring to ```skytap``` based lab environment that is not available any more in its original form.

You will still be able to perform all the tasks in the modernization guide using supplemtary instructions below.

**Note:** Tasks 1-8 below are performed locally.
Task 9 will require ICP access


The guide assumes you have access to a pre-built development workstation VM with all software prerequisites loaded into ~/PurpleCompute.
Instead you will load pre-reqs by yourself instead.
Ensure all software pre-requisites defined [here](mock-engagement.md) are installed.


Use the following supplementary instructions to complete the tasks described in the modernization guide above. You still follow the instructions provided in the guide, use the supplementary instruction below to address the logistal issues due to missing skytap VMs.


## Task 1: Analyze the application by using Transformation Advisor

Use local TA installed on your laptop [earlier](install-TA.md).
Upload to TA scan results ZIP file for Customer Order into TA Local. The ZIP file was generated from TA 1.4,but will still work.
https://ibm.ent.box.com/s/5sjn1kgn14x5avmp6v9k5dpr0kl78bg9

## Task 2: Download the application code and import it into Eclipse

Create ~/PurpleCompute on your laptop, and follow instructions to clone application git repo.
Eclipse doesn't have to be installed under ~/PurpleCompute.
You can and should use more recent Eclipse version (e.g.Photon).  There will be some UI deviations from the screenshots provided which are based on earlier Eclipse Neon version.


## Task 3: Clean up the development environment

Before starting this task, make sure you have WebSphere Liberty configured in Eclipse as a server.
https://developer.ibm.com/wasdev/downloads/liberty-profile-using-eclipse/

After step 6 you may see tons of java build errors in the CustomerOrderServiceTest project due to missing Apache Wink dependency.  Download apache wink jar http://archive.apache.org/dist/wink/1.4.0/ and import it into the right place as shown below:

![purple](static/purple-wink.png "purple")

More java build errors are caused by missing Jackson jars.  Locate and download missing jars e.g. jackson-core-asl-1.9.13.jar and jackson-jaxrs-1.9.13.jar in Maven repo https://mvnrepository.com
Import Jackson jars into EAR/lib directory:

![purple](static/purple-jackson.png "purple")

After the above, you should be in the exact same state in the Problems view as in the Step 7 of the guide.

Clear the XSLT Validation options (slightly different UI in Eclipse Photon)
![purple](static/purple-xslt.png "purple")

## Task 4: Re-create DB2 Datastore on IBM Cloud Private

On-premise DB2 backend is simulated by a  DB2 container with preloaded application database.
The DB2 container should be pre-deployed somewhere and available, DB2 endpoint should be provided. Currently (Nov 2018):

DB2 endpoint:
```
172.16.50.215:30475
```
**Note**:  172.16.50.*   is an internal lab environment accessible only via VPN

DB2 credentials:
```
DB2 userID  db2inst1
DB2 Password: passw0rd
```
**Note**: If no pre-deployed DB2 instance is avaialable and accessible, follow instructions in Appendix 1 below to deploy a copy of preloaded DB2 container on your own ICP instance.



## Task 5: Configure the Software Analyzer
no supplementary instructions required

## Task 6: Run the Software Analyzer
no supplementary instructions required

## Task 7: Configure the WebSphere Liberty Server

Download target server.xml directly to your laptop
[server.xml](solutions/purple-compute/server.xml)

Use locally installed Liberty instance on your laptop, which Eclipse points to:

![purple](static/purple-liberty.png "purple")


The downloaded server.xml may need to be tweaked in few places to reflect your environement (areas where change is likely needed is highlighted below)

![purple](static/purple-liberty-config.png "purple")



## Task 8: Run the application

The location of application EAR file and DB2 jdbc drivers consistent with the server.xml above is shown below:
![purple](static/purple-liberty-files.png "purple")

No other supplementary instructions required.

## Task 9: Deploy Customer Order application on ICP.

Follow the same steps used to deploy [Plants by WebSphere](plants-by-websphere.md) to ICP.
You can use
[deployment.json](solutions/purple-compute/deployment.json)
and
[service.json](solutions/purple-compute/service.json)


to deploy a container.

You can do it manually via ```kubectl create -f ```, but if you do it few times, you will quickly realize that for most part, the definitions of  Kubernetes deployment and a service are basically the same and follow a template with only a few variable parameters.
You can use a generic ```deploy-app``` helm chart to deploy any application provided [here](deploy-app)
The deploy-app helm chart implements a common deployment template, you will only need to provide few required parameters either in Values.yaml or via --set directive in command line
```
helm install ./deploy-app --name purple-compute-app  --set image.repository=mycluster.icp:8500/default/purple-compute-app --set service.servicePort=8080 --tls
```

Confirm the Helm release is deployed in ICP console, the pod has started, then test the JBoss application
