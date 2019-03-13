Steps to create tWAS vm with plants by websphere installed
***

Get the hostname and port for the plantsdb-preloaded DB2 container.  In our case it's
hostname: 9.42.30.105
port: 32145
db2 user: db2inst1
db2 password: db2Pa2359w0rd123


1) modify the wsadmin.py and replace those values if required

2) Build the docker container
`cd CreatePreConfiguredWASContainer`  
`docker build . -t twas-plantsbywebsphere`

That should be it.  You can now proceed to running the course. 
