
# Scalable Wordpress Deployment Using Bitnami Containers, Kubernetes and Google Cloud Platform

- [Prerequisites](#prerequisites)
  + [Container engine environment](#container-engine-environment)
- [Download the configuration files](#download-the-configuration-files)
- [Create the Docker container images](#create-the-docker-container-images)
    - [Wordpress Image](#wordpress-image)
    - [Apache Image](#apache-image)
- [Create your cluster](#create-your-cluster)
- [MariaDB](#mariadb)
  + [Create persistent disk](#create-persistent-disk)
  + [MariaDB master pod and service](#mariadb-master-pod-and-service)
  + [MariaDB slave pod and service](#mariadb-slave-pod-and-service)
- [Wordpress](#wordpress)
  + [Create Google cloud storage bucket](#create-google-cloud-storage-bucket)
  + [Wordpress secret store](#wordpress-secret-store)
  + [Wordpress pod and service](#wordpress-pod-and-service)
- [Apache](#apache)
  + [Apache pod and service](#apache-pod-and-service)
- [Access your Wordpress server](#access-your-wordpress-server)
- [Scaling the Wordpress blog](#scaling-the-wordpress-blog)
- [Take down and restart Wordpress](#take-down-and-restart-wordpress)
- [Cleanup](#cleanup)

This tutorial walks through setting up a scalable [Wordpress](http://wordpress.org) deployment on Google Container Engine using the [Bitnami Container Images](https://bitnami.com/docker) for [Docker](https://www.docker.com/). If you're just looking for the quickest way to get Wordpress up and running you might prefer our [prebuilt installers, VMs and Cloud Images](http://www.bitnami.com/stack/wordpress). If you're interested in getting hands on with [Kubernetes](http://kubernetes.io) and [Google Container Engine](https://cloud.google.com/container-engine/), read on....

The following illustration provides an overview of the architecture we'll setup using Kubernetes and Bitnami container images for our Wordpress deployment.

![Architecture](images/architecture.png)

We'll be creating a scalable Wordpress deployment backed by a cluster of MariaDB instances which can be scaled horizontally on-demand. We'll attach a persistent disk to the MariaDB master instance so that our database backend can preserve its state across shutdown and startup. Three replicas of the Apache container will act as the frontend to our Wordpress deployment. For high availability the Wordpress instances will be configured to have three replicas. We also configure load balancing, external IP, a secret store and health checks.

## Prerequisites

### Container engine environment

Set up your Google Container Engine environment using [these instructions](https://cloud.google.com/container-engine/docs/before-you-begin).

## Download the configuration files

Clone the [bitnami-docker](https://github.com/bitnami/bitnami-docker) GitHub repository. The files used in this tutorial can be found in the `gke/wordpress` directory of the cloned repository:

```bash
$ git clone https://github.com/bitnami/bitnami-docker.git
$ cd bitnami-docker/gke/wordpress
```

## Create the Docker container images

In this section we'll build the Docker images for our Wordpress blog.

### Wordpress image

The Wordpress image is built using the `Dockerfile` from the `dockerfiles/wordpress-php` directory. Docker container images can extend from other existing images. Since Wordpress is a PHP application, we'll extend the [bitnami/php-fpm](https://hub.docker.com/r/bitnami/php-fpm/) image.

In the `Dockerfile` the Wordpress source is copied into the `/app` directory of the container. The base `bitnami/php-fpm` image uses [s6-overlay](https://github.com/just-containers/s6-overlay) for process supervision. We use the infrastucture provided by s6-overlay to create a container initialization script `/etc/cont-init.d/60-wordpress` which configures the database connection parameters for Wordpress in `/app/wp-config.php` among other things.

Our Wordpress image uses the patched versions of the [Amazon Web Services](https://github.com/timwhite/wp-amazon-web-services) and [WP Offload S3](https://github.com/timwhite/wp-amazon-s3-and-cloudfront) plugins for use with Google cloud storage. It also installs the [HyperDB](https://github.com/taskrabbit/makara) plugin to enable support for our database replication backend.

Build the image by running:

```bash
$ cd dockerfiles/wordpress-php/
$ docker build -t gcr.io/<google-project-name>/wordpress-php .
```

Then push this image to the Google Container Registry:

```bash
$ gcloud docker push -- gcr.io/<google-project-name>/wordpress-php
```

### Apache image

The Apache image is built using the `Dockerfile` from the `dockerfiles/wordpress-apache` directory and it extends the [bitnami/apache](https://hub.docker.com/r/bitnami/apache/) image.

Apache serves as the frontend of our deployment and handles client HTTP requests while delegating the PHP processing to the `PHP-FPM` daemon of the Wordpress container instances.

The Wordpress application and plugins source is also copied into the Apache image at `/app` so that the static site assets (css, js, images, etc) are locally available and ready to be served by the Apache server.

Like the Wordpress image, we use a container initialization script `/etc/cont-init.d/60-wordpress` to create a catch-all virtual host configuration with `/app` as the [DocumentRoot](https://httpd.apache.org/docs/2.4/mod/core.html#documentroot) and the Wordpress instances as the PHP processing backends.

Build the image by running:

```bash
$ cd dockerfiles/wordpress-apache/
$ docker build -t gcr.io/<google-project-name>/wordpress-apache .
```

Then push this image to the Google Container Registry:

```bash
$ gcloud docker push -- gcr.io/<google-project-name>/wordpress-apache
```

## Create your cluster

Now you are ready to create the Kubernetes cluster on which you'll run the Wordpress deployment. A cluster consists of a master API server hosted by Google and a set of worker nodes.

Create a cluster named `wordpress`:

```bash
$ gcloud beta container clusters create wordpress
```

A successful create response looks like:

```
Creating cluster wordpress...done.
Created [.../projects/docker-opensource/zones/us-central1-b/clusters/wordpress].
kubeconfig entry generated for wordpress.
NAME       ZONE           MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NUM_NODES  STATUS
wordpress  us-central1-b  1.1.3           104.154.43.166  n1-standard-1  3          RUNNING

```

Now that your cluster is up and running, we are set to launch the components that make up our deployment.

## MariaDB

![MariaDB](images/mariadb.png)

The above diagram illustrates our MariaDB backend. We'll create a MariaDB master/slave configuration where the slave pods will replicate the Wordpress database from the master. This will enable us to horizontally scale the MariaDB slave pods when required. A persistent disk attached to the MariaDB master instance will allow the database backend to preserve its state across startup and shutdown.

### Create persistent disk

We'll make use of [volumes](http://kubernetes.io/v1.0/docs/user-guide/volumes.html) to create a persistent disk for the MariaDB master. This volume is used in the pod definition of the MariaDB master controller.

Create the persistent disk using:

```bash
$ gcloud compute disks create --size 200GB mariadb-disk
Created [.../projects/docker-opensource/zones/us-central1-b/disks/mariadb-disk].
NAME         ZONE          SIZE_GB TYPE        STATUS
mariadb-disk us-central1-b 200     pd-standard READY
```

### MariaDB master pod and service

The first thing that we're going to do is start a [pod](http://kubernetes.io/v1.0/docs/user-guide/pods.html) for MariaDB master. We'll use a [replication controller](http://kubernetes.io/v1.0/docs/user-guide/replication-controller.html) to create the pod — even though it's a single pod, the controller is still useful for monitoring health and restarting the pod if required.

We'll use the config file `mariadb-master-controller.yml` for the pod which creates a single MariaDB master pod with the label `name=mariadb-master`. The pod uses the [bitnami/mariadb](https://hub.docker.com/r/bitnami/mariadb/) image and specifies the user and database to create as well as the replication parameters using environment variables.

> **Note**:
>
> You should change the value of the `MARIADB_PASSWORD` and `MARIADB_REPLICATION_PASSWORD` env variables to your choosing.

To create the pod:

```bash
$ kubectl create -f mariadb-master-controller.yml
```

Check to see if the pod is running. It may take a minute to change from `Pending` to `Running`:

```bash
$ kubectl get pods -l name=mariadb-master
NAME                   READY     STATUS    RESTARTS   AGE
mariadb-master-ze6xc   1/1       Running   0          39s
```

A [service](http://kubernetes.io/v1.0/docs/user-guide/services.html) is an abstraction which defines a logical set of pods and a policy by which to access them. It is effectively a named load balancer that proxies traffic to one or more pods.

When you set up a service, you tell it the pods to proxy based on pod labels. The pod that you created in previous step has the label `name=mariadb-master`.

We'll use the file `mariadb-master-service.yml` to create a service for the MariaDB master pod. The `selector` field of the service configuration determines which pods will receive the traffic sent to the service. So, the configuration specifies that we want this service to point to pods labeled with `name=mariadb-master`.

Start the service:

```bash
$ kubectl create -f mariadb-master-service.yml
```

See it running:

```bash
$ kubectl get services mariadb-master
NAME             CLUSTER_IP       EXTERNAL_IP   PORT(S)    SELECTOR              AGE
mariadb-master   10.247.244.249   <none>        3306/TCP   name=mariadb-master   4s
```

### MariaDB slave pod and service

Next we setup the MariaDB slave pods and service. The slave pods will connect to the master service and replicate the Wordpress database. The `mariadb-slave-controller.yml` config file describes the slave pods and specifies 3 replicas with the label `name=mariadb-slave`.

> **Note**: You should change the value of the `MARIADB_PASSWORD`, `MARIADB_REPLICATION_PASSWORD` and `MARIADB_MASTER_PASSWORD` env variables with the ones specified in `mariadb-master-controller.yml`

To create the pod:

```bash
$ kubectl create -f mariadb-slave-controller.yml
```

Check to see if the pod is running. It may take a minute to change from `Pending` to `Running`:

```bash
$ kubectl get pods -l name=mariadb-slave
NAME                  READY     STATUS    RESTARTS   AGE
mariadb-slave-85jrd   1/1       Running   0          16s
mariadb-slave-hfcrb   1/1       Running   0          16s
mariadb-slave-rj1bo   1/1       Running   0          16s
```

As with the MariaDB master pod, we want a service to group the slave pods. We'll use the file `mariadb-slave-service.yml` to create a service which specifies the label `name=mariadb-slave` as the pod `selector`.

Start the service:

```bash
$ kubectl create -f mariadb-slave-service.yml
```

See it running:

```bash
$ kubectl get services mariadb-slave
NAME            CLUSTER_IP      EXTERNAL_IP   PORT(S)    SELECTOR             AGE
mariadb-slave   10.247.247.10   <none>        3306/TCP   name=mariadb-slave   6s
```

## Wordpress

Now that we have our database backend up and running, lets set up the Wordpress application instance.

![Wordpress](images/wordpress.png)

The above diagram illustrates the Wordpress pod and service configuration.

### Create Google cloud storage bucket

To allow horizontal scaling of the Wordpress blog we'll use the Google cloud storage service, in S3 interoperability mode, to host files uploaded to the Wordpress media library. This also ensures that the uploaded files are persistent across pod startup and shut down as you will see in [Take down and restart Wordpress](#take-down-and-restart-wordpress).

For Wordpress to be able to access google cloud storage, we need to provide the access credentials to our Wordpress pod.

To create a bucket and developer key:

  1. Go to the [Google Developers Console](https://console.developers.google.com/).
  2. Click the name of your project.
  3. In the left sidebar, go to **Storage > Cloud Storage > Browser**.
  4. Select **Create bucket** and give it the name.

  ![Create Bucket](images/create-bucket.png)

  5. In the left sidebar, go to **Storage > Cloud Storage > Storage settings**.
  6. Select **Interoperability**.
  7. If you have not set up interoperability before, click **Enable interoperability access**.
  8. Click **Create a new key**.

  ![Create Developer Key](images/create-developer-key.png)

Make a note of the generated **Access Key** and **Secret**, in the next section we'll specify them in the secrets definition.

### Wordpress secret store

A [secret key store](http://kubernetes.io/v1.0/docs/user-guide/secrets.html) is intended to hold sensitive information such as passwords, access keys, etc. Having this information in a key store is safer and more flexible then putting it in to the pod definition.

We'll create a key store to save the sensitive configuration parameters of our deployment. This includes, but is not limited to the database password, session tokens, cloud storage access key id and secret.

Begin by encoding our secrets in base64, starting with the database password.

```bash
$ base64 -w128 <<< "secretpassword"
c2VjcmV0cGFzc3dvcmQK
```

Next, we encode the S3 credentials as generated in [Create Google cloud storage bucket](#create-google-cloud-storage-bucket).

```bash
$ base64 <<< "GOOGUF56OWN3R3LFYOZE"
R09PR1VGNTZPV04zUjNMRllPWkUK

$ base64 <<< "A+uW0XLz9Y+EHUGRUf1V2uApcI/TenhBtUnPao7i"
QSt1VzBYTHo5WStFSFVHUlVmMVYydUFwY0kvVGVuaEJ0VW5QYW83aQo=
```

To secure Wordpress we need to generate random and unique hashes for each of the following Wordpress parameters `AUTH_KEY`, `SECURE_AUTH_KEY`, `LOGGED_IN_KEY`, `NONCE_KEY`, `AUTH_SALT`, `SECURE_AUTH_SALT`, `LOGGED_IN_SALT` and `NONCE_SALT`. Generate a random hash for each of these parameters (8 in total) and encode them using `base64`.

```bash
$ base64 -w128 <<< "mCjVXBV6jZVn9RCKsHZFGBcVmpQd8l9s"
bUNqVlhCVjZqWlZuOVJDS3NIWkZHQmNWbXBRZDhsOXMK
```

> **Tip**:  You can use `pwgen -csv1 64` to generate a random and unique 64 character hash value.

> **Pro Tip**: To generate a random hash and encode it with `base64` in a single command use `base64 -w128 <<< $(pwgen -csv1 64)`

Update `wordpress-secrets.yml` with the `base64` encoded database password, S3 credentials and hashes and create the secret key store using:

```bash
$ kubectl create -f wordpress-secrets.yml
```

See it running:

```bash
$ kubectl get secrets -l name=wordpress-secrets
NAME                TYPE      DATA      AGE
wordpress-secrets   Opaque    11        3s
```

This secret key store will be mounted at `/etc/secrets` in read-only mode in the Wordpress pods.

### Wordpress pod and service

The controller and its pod template is described in the file `wordpress-controller.yml`. It specifies 3 replicas of the pod with the label `name=wordpress-php`.

> **Note**:
>
> Change the image name to `gcr.io/<google-project-name>/wordpress-php` as per the build instructions in [Wordpress image](#wordpress-image).

Using this file, you can start your Wordpress controller with:

```bash
$ kubectl create -f wordpress-controller.yml
```

Check to see if the pods are running. It may take a few minutes to change from `Pending` to `Running`:

```bash
$ kubectl get pods -l name=wordpress-php
NAME                  READY     STATUS    RESTARTS   AGE
wordpress-php-iwubl   1/1       Running   0          18s
wordpress-php-prrcf   1/1       Running   0          18s
wordpress-php-xoghd   1/1       Running   0          18s
```

We want a service to group the Wordpress pods. The service specification for the Wordpress service is defined in `wordpress-service.yml` and specifies the label `name=wordpress-php` as the pod `selector`.

Start the service using:

```bash
$ kubectl create -f wordpress-service.yml
```

See it running:

```bash
$ kubectl get services wordpress-php
NAME            CLUSTER_IP       EXTERNAL_IP   PORT(S)    SELECTOR             AGE
wordpress-php   10.247.254.201   <none>        9000/TCP   name=wordpress-php   3s
```

## Apache

Now that we have the MariaDB and Wordpress pods up and running, lets set up the Apache pods and service which will act as the frontend to our Wordpress deployment as illustrated in the following illustration.

![Apache](images/apache.png)

### Apache pod and service

The controller and its pod template is described in the file `apache-controller.yml`. It specifies 3 replicas of the server with the label `name=wordpress-apache`.

> **Note**
>
> 1. Change the image name to `gcr.io/<google-project-name>/wordpress-apache` as per the build instructions in [Apache image](#apache-image).

Using this file, you can start the Apache controller with:

```bash
$ kubectl create -f apache-controller.yml
```

Check to see if the pods are running:

```bash
$ kubectl get pods -l name=wordpress-apache
NAME                     READY     STATUS    RESTARTS   AGE
wordpress-apache-qiw1h   1/1       Running   0          22s
wordpress-apache-so2oj   1/1       Running   0          22s
wordpress-apache-uq52m   1/1       Running   0          22s
```

Once the servers are up, you can list the pods in the cluster, to verify that they're all running:

```bash
$ kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
mariadb-master-ze6xc     1/1       Running   0          4m
mariadb-slave-85jrd      1/1       Running   0          2m
mariadb-slave-hfcrb      1/1       Running   0          2m
mariadb-slave-rj1bo      1/1       Running   0          2m
wordpress-apache-qiw1h   1/1       Running   0          36s
wordpress-apache-so2oj   1/1       Running   0          36s
wordpress-apache-uq52m   1/1       Running   0          36s
wordpress-php-iwubl      1/1       Running   0          1m
wordpress-php-prrcf      1/1       Running   0          1m
wordpress-php-xoghd      1/1       Running   0          1m
```

You'll see a single MariaDB master pod, three MariaDB slave pods, three Wordpress pods and three Apache pods. In [Scaling the Wordpress blog](#scaling-the-wordpress-blog) we'll see how we can scale the MariaDB slave, Wordpress and Apache pods to meet the growing demands of your blog.

As with the other pods, we want a service to group the Apache pods. However, this time it's different: this service is user-facing, so we want it to be externally visible. That is, we want a client to be able to request the service from outside the cluster. To accomplish this, we can set the `type: LoadBalancer` field in the service configuration.

The service specification for the Apache is in `apache-service.yml` which specifies the label `name=wordpress-apache` as the pod `selector`.

```bash
$ kubectl create -f apache-service.yml
```

See it running:

```bash
$ kubectl get services wordpress-apache
NAME               CLUSTER_IP      EXTERNAL_IP   PORT(S)   SELECTOR                AGE
wordpress-apache   10.247.244.54                 80/TCP    name=wordpress-apache   7s
```

## Access your Wordpress server

Now that the firewall is open, you can access the service over the internet. Find the external IP of the Apache service you just set up using:

```bash
$ kubectl describe services wordpress-apache
Name:                 wordpress-apache
Namespace:            default
Labels:               name=wordpress-apache
Selector:             name=wordpress-apache
Type:                 LoadBalancer
IP:                   10.247.244.54
LoadBalancer Ingress: 104.197.91.177
Port:                 <unnamed> 80/TCP
NodePort:             <unnamed> 31959/TCP
Endpoints:            10.244.0.7:80,10.244.1.7:80,10.244.2.6:80
Session Affinity:     None
Events:
  FirstSeen LastSeen  Count From      SubobjectPath Reason      Message
  ───────── ────────  ───── ────      ───────────── ──────      ───────
  1m    1m    1 {service-controller }     CreatingLoadBalancer  Creating load balancer
  22s   22s   1 {service-controller }     CreatedLoadBalancer Created load balancer
```

Visit `http://x.x.x.x` in your favourite web browser, where `x.x.x.x` is the IP address listed next to `LoadBalancer Ingress` in the response of the above command. You will be greeted with the Wordpress setup:.

Once you complete the setup, we need to enable the Amazon AWS Services and WP Offload S3 plugins from the Wordpress administration panel.

  1. Login to the administration panel
  2. On the left sidebar click on **Plugins**
  3. Activate the **Amazon Web Services** and **WP Offload S3** plugins
  4. Load the **Settings** panel of **WP Offload S3**
  5. Enter the bucket name created in [Create Google cloud storage bucket](#create-google-cloud-storage-bucket)
  6. Enable the **Remove Files From Server** configuration
  7. Save the Changes

You now have a scalable Wordpress deployment. The next section demonstrates how it can be scaled without any downtime.

## Scaling the Wordpress blog

Since the MariaDB slave, Wordpress and Apache pods are defined as services that use a replication controller, you can easily resize the number of pods in the replication controller as follows:

To scale the MariaDB slave pods:

```bash
$ kubectl scale --replicas=5 rc mariadb-slave
```

The configuration for the controllers will be updated, to specify that there should be 5 replicas running. The replication controller adjusts the number of pods it is running to match that, and you will be able to see the additional pods running:

```bash
$ kubectl get pods -l name=mariadb-slave
NAME                  READY     STATUS    RESTARTS   AGE
mariadb-slave-85jrd   1/1       Running   0          8m
mariadb-slave-hfcrb   1/1       Running   0          8m
mariadb-slave-kemgr   1/1       Running   0          12s
mariadb-slave-kzr6y   1/1       Running   0          12s
mariadb-slave-rj1bo   1/1       Running   0          8m
```

Similarly to scale the Wordpress pods:

```bash
$ kubectl scale --replicas=5 rc wordpress-php
```

and to scale the Apache pods:

```bash
$ kubectl scale --replicas=5 rc wordpress-apache
```

You can scale down in the same manner.

> **Note**: The MariaDB master controller cannot be scaled.

## Take down and restart Wordpress

Because we used a persistent disk for the MariaDB master pod and used Google cloud storage for files uploaded to the Wordpress media library, the state of your Wordpress deployment is preserved even when the pods it's running on are deleted. Lets try it.

```bash
$ kubectl delete rc wordpress-apache wordpress-php mariadb-slave mariadb-master
```

*Deleting the replication controller also deletes its pods.*

Confirm that the pods have been deleted:

```bash
$ kubectl get pods
```

Then re-create the pods:

```bash
$ kubectl create -f mariadb-master-controller.yml
$ kubectl create -f mariadb-slave-controller.yml
$ kubectl create -f wordpress-controller.yml
$ kubectl create -f apache-controller.yml
```

Once the pods have restarted, the `mariadb-master`, `mariadb-slave`, `wordpress-php` and `wordpress-apache` services pick them up immediately based on their labels, and your Wordpress blog is restored.

## Cleanup

To delete your application completely:

*If you intend to teardown the entire cluster, jump to __Step 4__.*

  1. Delete the controllers:

  ```bash
  $ kubectl delete rc wordpress-apache wordpress-php mariadb-slave mariadb-master
  ```

  2. Delete the services:

  ```bash
  $ kubectl delete service wordpress-apache wordpress-php mariadb-slave mariadb-master
  ```

  3. Delete the secret key store

  ```bash
  $ kubectl delete secret wordpress-secrets
  ```

  4. Delete your cluster:

  ```bash
  $ gcloud beta container clusters delete wordpress
  ```

  5. Delete the firewall rule:

  ```bash
  $ gcloud compute firewall-rules delete wordpress-http
  ```

  6. Delete the disks:

  ```bash
  $ gcloud compute disks delete mariadb-disk
  ```

  7. Delete the bucket and developer key from the [Google Developers Console](https://console.developers.google.com/)
