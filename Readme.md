# Wordpress installation on Kubernetes

Here is a detailed step by step how to install wordpress in local minikube cluster with WP and MySQL. Use this as a simple tutorial to cover basic concepts of Kubernetes setup and usage. This has been my notebook to document learning process of various steps to do when starting with Kubernetes.

### Table of contents

1. [Prerequisites](#prerequisites)
2. [Where to start](#where%20to%20start)
3. [Create secrets.yml](#k8s%20secrets)
4. [Config map for MySQL](#configmap)
5. [Deployment file](#create%20deployment%20file)
6. [Service](#service)

### Prerequisites

This tutorial assumes you have access to a cluster or have a local cluster installed and running. One of the options is [minikbe](https://minikube.sigs.k8s.io/docs/start/) which has been working for me without issues.

### Where to start

First let's think about what containers do we need to create. We will need one container to run our database (wordpress backend) and one for wordpress itself. [Docker Hub](https://hub.docker.com) will be our source of images.

### k8s Secrets

Since we know we will connect to MySQL database we need to create secrets in k8s to store our MySQL DB password.
Create `secrets.yml` to store MySQL database password and insert appropriate code in the file. Don't forget to base64 encode the password.

Then run `kubectl apply -f secrets.yml` to create kubernetes secret. You can verify the operation by typing `kubectl get secrets`

### Configmap

A ConfigMap is an API object used to store non-confidential data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume. In our case we will be using MySQL version 8.0.23 where, compared to 5.x version, the default authentication has been changed. We need to specify this in ConfigMap:

```
# config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config ## name of ConfigMap; this will be referred from volume definition in deployment file
  labels:
    app: mysql
data:
  ## default_auth is the name of config. This will be referred from volume mount definition subpath
  default_auth: |
    [mysqld]
    default_authentication_plugin=mysql_native_password
```

Put this code into `config-map.yml` file and to apply config map run `kubectl apply -f config-map.yml` You can cbeck what config maps you have defined by running `kubectl get cm`
Alternatively we can just pass this as `args` from deployment file.

### Persistent volume and persistent volume claim

Our next step is to define persistent volume to make sure that our database has a persistent location to store the data. This is essential to make sure that when mysql container is restarted (or pods/cluster are restarted) we will not lose the data we have created. It is accomplished by creating 'persistent storage' in our cluster.

Persistent volume will define and allocate storage space we can consume from, while persistent volume claim will be used to 'claim' a portion of this storage for our database. PV will allow to store data outside the container and attach a volume to the container that will exist also when container stops.

Create `pers_volume.yml' and put in the appropriate code:

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-pv
  labels:
    type: local
spec:
  capacity:
    storage: "2Gi" # reserve 2GB of storage
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  hostPath:
    path: "data/wordpress-pv"
```

The run `kubectl apply -f pers_volume.yml` to create persistent volume.

Now we need to create persistent volume claim to allocate certain space for our database. Create `pv_claim.yml` file and insert appropriate code, then run `kubectl apply -f pv_claim.yml` to create PV claim.

### Create deployment file

Two containers will be needed in `deployment.yml`:

1. wordpress
2. mysql

Create `deployment.yml` which will accomplish this. To create the actual deployment you can run `kubectl apply -f deployment.yml` and wait for the containers to create and start.

### Service

This is the final step in the process which will expose WP container outside of the cluster. For this we need to create a service using `service.yml` and the `kubectl apply -f service.yml` - this will expose a port that we can access outside of our minikube cluster.

The final step will be to run `minikube service wordpress-service` and wait the browser to open on assigned url and port. 

You are now ready to start using wordpress installation running in your kubernetes cluster.