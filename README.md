# helm-oci-mds-wordpress-nfs

[![License: UPL](https://img.shields.io/badge/license-UPL-green)](https://img.shields.io/badge/license-UPL-green) [![Quality gate](https://sonarcloud.io/api/project_badges/quality_gate?project=oracle-devrel_terraform-oci-arch-ci-cd)](https://sonarcloud.io/dashboard?id=oracle-devrel_terraform-oci-arch-ci-cd)


## Introduction

This Helm chart will bootstrap a WordPress deployment using a MySQL Database Systems (MDS) as the database in a Kubernetes cluster deployed in Oracle CLoud Infrastrcuture (OCI). It will also use OCI filesystems as the persistent storage for the wordpress deployment to allow multiple pods for high availability. It will create a loadbalancer with an externally accessible IP address to provide access to the Wordpress application. The Kubernetes cluster can be a cluster deployed using Oracle Container Engine for Kuberntes (OKE), or it can be a customer managed cluster deployed on virtual machine instances. 


This Helm chart relies on the OCI Service Operator for Kubernetes (OSOK) and it is a pre-requisite to have OSOK deployed within the cluster to use this Helm chart.


## Pre-requisites

- A Kuberntes cluster deployed in OCI 
- [OCI Service Operator for Kuberntes (OSOK) deployed in the cluster](https://github.com/oracle/oci-service-operator/blob/main/docs/installation.md)
- [kubectl installed and using the context for the Kubernetes cluster where the MDS resource will be deployed](https://kubernetes.io/docs/tasks/tools/)
- [Helm installed](https://helm.sh/docs/intro/install/)
- [OCI File System created](https://docs.oracle.com/en-us/iaas/Content/File/Tasks/creatingfilesystems.htm) and get server IP and mountTargetID OCID


##  Getting Started

**1. Clone or download the contents of this repo** 
     
     git clone https://github.com/chiphwang1/helm-oci-mds-wordpress.git

**2. Change to the directory that holds the Helm Chart** 

      cd ./helm-oci-mds-wordpress

**3. Populate the values.yaml file with information to deploy the MDS resource**


**4. Create the namespace where the MDS resource will be deployed**

     kubectl create ns <namespace name>

**5. Install the Helm chart. Best practice is to assign the username and password during the installation of the Helm chart instead of adding it to the values.yam file.**

     helm -n <namespace name> install \
     --set database.username=<database password> \  
     --set database.password=<database password> \
       <name for this install> .
  
  **Example:**
  helm -n wordpress --set database.username=admin  --set database.password=Admin12345 ocimds .
  
 The password must be between 8 and 32 characters long, and must contain at least 1 numeric character, 1 lowercase character, 1 uppercase character, and 1   special (nonalphanumeric) character.


**6. List the Helm installation**

     helm  -n <namespace name> ls


**7. To uninstall the Helm chart**

     helm uninstall -n <namespace name> <name of the install> .
     
     
   **Important Note**
 
 Uninstalling the helm chart will only remove the mysqldbsystem resource from the cluster and not from OCI. You will need to use the console or the OCI cli to remove the MDS from OCI. This is to prevent accidental deletion of the database.

     
  **Notes/Issues:**
 
 Provisioning the MySQL Database Systems (MDS) can take up to 20 minutes. The Wordpress deployment will not be available until all components are up. 
 
 To confirm that the MDS is active run the following command and check the status of the MDS system.

```sh
$ kubectl -n wordpress get mysqldbsystems -o wide
NAME    DISPLAYNAME   STATUS   OCID                                                                                       AGE
MDS       MDS         Active   ocid1.mysqldbsystem.oc1.iad.aaaaaaaapgrgv23wlrf47nvlp26w6nvfujntvimjkdmw6jo5eft3lo7j5s6q   15h
```

It is also possible to monitor the status of the MDS creation by observing the logs of the dbjob container. To get the name of the db contianer run the following command. 

```sh
$ kubectl -n wordpress get pods
NAME                         READY   STATUS     RESTARTS   AGE
dbjob-khr9n                  1/1     Running    0          2m10s
wordpress-844ffbb9d7-md744   0/1     Init:0/1   0          2m10s
```

To watch the logs use the following command. The logs will update every 5 seconds until the MDS is ready.

```sh
$ kubectl -n wodpress logs -f dbjob-khr9n

--------------------------------
Check for readiness of the MySQL instance by checking for creation of secret mysql
NAME                            TYPE                                  DATA   AGE
default-token-6qpqr             kubernetes.io/service-account-token   3      20m
internal-kubectl-token-h6jp8    kubernetes.io/service-account-token   3      19m
mysqlsecret                     Opaque                                2      19m
sh.helm.release.v1.mysqldb.v1   helm.sh/release.v1                    1      19m
--------------------------------
Check for readiness of the MySQL instance by checking for creation of secret mysql
NAME                            TYPE                                  DATA   AGE
default-token-6qpqr             kubernetes.io/service-account-token   3      20m
internal-kubectl-token-h6jp8    kubernetes.io/service-account-token   3      19m
mysqlsecret                     Opaque                                2      19m
sh.helm.release.v1.mysqldb.v1   helm.sh/release.v1                    1      19m
--------------------------------
--------------------------------
MySQL instance ready
--------------------------------
--------------------------------
Connecting to Database instance
--------------------------------
Database wordpress created 
--------------------------------
NAME        TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
wordpress   LoadBalancer   10.96.12.177   129.158.42.147   80:32131/TCP   22m

```


 
 To retreive the IP address of the loadbalancer use the following command.
 
 ```sh
$ kubectl -n wordpress get svc
NAME        TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)       AGE
wordpress   LoadBalancer   10.96.12.177  129.158.42.147   80:30388/TCP    22m
```
 

## MySQL DB System value.yaml Specification


| Parameter                          | Description                                                         | Type   | Mandatory |
| ---------------------------------- | ------------------------------------------------------------------- | ------ | --------- |
| `name` | name of MySQL DB System resource. This is name that is used in the Kubernetes cluster | string | yes  |
| `spec.displayName` | The user-friendly name for the Mysql DbSystem. The name does not have to be unique. | string | yes     |
| `spec.compartmentId` | The [OCID](https://docs.cloud.oracle.com/Content/General/Concepts/identifiers.htm) of the compartment of the Mysql DbSystem. | string | yes       |
| `spec.shapeName` | The name of the shape. The shape determines the resources allocated  | string | yes       |
| `spec.subnetId` | The OCID of the subnet the DB System is associated with.  | string | yes       |
| `spec.dataStorageSizeInGBs`| Initial size of the data volume in GBs that will be created and attached. Keep in mind that this only specifies the size of the database data volume, the log volume for the database will be scaled appropriately with its shape. | int    | yes       |
| `spec.isHighlyAvailable` | Specifies if the DB System is highly available.  | boolean | yes       |
| `spec.availabilityDomain`| The availability domain on which to deploy the Read/Write endpoint. This defines the preferred primary instance. | string | yes        |
| `spec.faultDomain`| The fault domain on which to deploy the Read/Write endpoint. This defines the preferred primary instance. | string | no        |
| `spec.configuration.id` | The OCID of the Configuration to be used for this DB System. [More info about Configurations](https://docs.oracle.com/en-us/iaas/mysql-database/doc/db-systems.html#GUID-E2A83218-9700-4A49-B55D-987867D81871)| string | yes |
| `spec.description` | User-provided data about the DB System. | string | no |
| `spec.hostnameLabel` | The hostname for the primary endpoint of the DB System. Used for DNS. | string | no |
| `spec.mysqlVersion` | The specific MySQL version identifier. | string | no |
| `spec.port` | The port for primary endpoint of the DB System to listen on. | int | no |
| `spec.portX` | The TCP network port on which X Plugin listens for connections. This is the X Plugin equivalent of port. | int | no |
| `spec.ipAddress` | The IP address the DB System is configured to listen on. A private IP address of your choice to assign to the primary endpoint of the DB System. Must be an available IP address within the subnet's CIDR. If you don't specify a value, Oracle automatically assigns a private IP address from the subnet. This should be a "dotted-quad" style IPv4 address. | string | no |
| `database.username` | The admin username for the administrative user for the MuSQL DB Systesm. This should be assigned during the deployment of the Helm chart and not kept in the values.yaml file| string | yes       |
| `database.password` | The admin password for Mysql DB System. The password must be between 8 and 32 characters long, and must contain at least 1 numeric character, 1 lowercase character, 1 uppercase character, and 1 special (nonalphanumeric) character. | string | yes       |
| `DBName` |  name of the database to create. | string | yes    |
| `mountTargetID` |  OCID of OCI File System Mount | string | yes    |
| `nfsServerIP` |  IP of OCI File System | string | yes    |
| `nfsPath` |  Path of OCI File System | string | yes    |



## Useful commands 


**1. To check the status of the MySQL DB System run the following command**
     
     kubectl -n <namespace of mysqldbsystem> get mysqldbsystems -o wide

**2. To describe the MySQL DB System run the following command** 
     
     kubectl -n <namespace of mysqldbsystem> describe mysqldbsystems <name of mysqldbsystem>

**3. To retreive the OCID of MySQL DB System run the following command** 

      kubectl -n <namespace of mysqldbsystem> get mysqldbsystems <name of mysqldbsystem> -o jsonpath="{.items[0].status.status.ocid}"
 
**4. To retrive the admin username of the MySQL DB System run the following command**
     
     kubectl -n  <namespace of mysqldbsystem>  get secret mysqlsecret -o  jsonpath="{.data.username}" | base64 --decode

**5. To retrive the admin password of the MySQL DB System run the following command**
     
     kubectl -n  <namespace of mysqldbsystem>  get secret mysqlsecret -o  jsonpath="{.data.password}" | base64 --decode

**6. To retreive endpoint information for the MySQL DB System run the following command**
     
     kubectl -n <namespace of mysqldbsystem> get secret <name of the MySQL DB System>  -o jsonpath='{.data.Endpoints}' | base64 --decode



## Additional Resources

- [OCI Service Operator for Kuberntes (OSOK) deployed in the cluster](https://github.com/oracle/oci-service-operator)
- [MySQL SB System Service](https://www.oracle.com/mysql/)
- [OCI File Storage](https://docs.oracle.com/en-us/iaas/Content/File/Concepts/filestorageoverview.htm)


## License
Copyright (c) 2021 Oracle and/or its affiliates.

Licensed under the Universal Permissive License (UPL), Version 1.0.

See [LICENSE](LICENSE) for more details.
