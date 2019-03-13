Create Pre-loaded DB2 instance
***

1) `docker pull ibmcom/db2express-c:latest`

2) Run container
`docker run -e LICENSE=accept -e DB2INST1_PASSWORD=db2Pa2359w0rd123 -p 50000:50000 --name plantsdb-preloaded ibmcom/db2express-c:latest db2start`

3) Log into container.  
`docker exec -it plantsdb-preloaded bash`

4) Change user to db2inst1  
`su - db2inst1`  

5) Copy TABLE.ddl to machine using vi
`vi TABLE.ddl`
Copy and paste contents of TABLE.ddl here into editor and save (esc :wq)


6) Create database  
`db2 create db PLANTSDB`

7) Connect to Database  
`db2 connect to PLANTSDB`

8) Run TABLE.DDL
`db2 -tvf TABLE.ddl`

9) To populate the database I couldn't figure out how to do it from the command line, when I copied to data to the machine it was too large to run using db2 -tvf <filename> so I did it using a graphical user interface (razorSQL), I then imported each table using the `.sql` files included here.  These were exported from an existing DB instance we had running somewhere else.  I'm not a DB guy, so this is as far as I got. Any experts who want to educate me on better ways to do this, please message me.  
They are `GO` separated, which is a question the importer asks.


10) Commit the image, as it now has the date we want on it.
`docker commit plantsdb-preloaded mycluster.icp:8500/default/plantsdb-preloaded:latest`

If you haven't pushed anything to ICP before, you may have to modify your hosts file and add certificates to you environment.  Here are the instructions for MAC:
https://github.ibm.com/vandepol/howto/blob/master/Docker/Certificates_Mac.md

11) Push this to ICP
`docker login mycluster.icp:8500`
provide username and password (usually admin/admin)

DEPLOY ON ICP

Login to mycluster.icp:8443 with username and PASSWORD
Configure commandline to connect to ICP

Run the following commands to deploy the images
`kubectl create -f deployment.json`
`kubectl create -f service.json`

To determine the port to access the database you can look on the ICP services page
Which should be located here:
https://mycluster.icp:8443/console/access/services/default/plantsdb-preloaded

or run
`kubectl get services`
Then search for plantsdb-preloaded
```
plantsdb-preloaded                       NodePort    10.0.0.140   <none>        50000:32145/TCP                                                                        2m
                                                      1d
```

In this case we would use: 9.42.30.105:32145


Next we create the tWAS VM
