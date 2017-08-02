# Introduction

In this tutorial, we will use Bitnami containers to run the <a href="http://www.subrion.org/" target="_blank">Subrion Open Source CMS</a>, taking advantage of Docker features such as linking and volumes.

### [MariaDB](https://github.com/bitnami/bitnami-docker-mariadb)

MariaDB will be used as the database server of choice and will be used by Subrion to store its database schema.

### [PHP-FPM](https://github.com/bitnami/bitnami-docker-php-fpm)

Subrion is a php application so we will use the PHP-FPM container to provide php support in the application stack.

### [Apache](https://github.com/bitnami/bitnami-docker-apache)

The Apache web server will be used to serve the Subrion application, with the php processing tasks delegated to the PHP-FPM container.

# Setting up Subrion

### Step 1: Download Subrion

Create a new project directory that will contain our app's content.

```bash
mkdir docker-subrion
cd docker-subrion
```

The latest version of Subrion can be downloaded from <a href="http://www.subrion.org/download/" target="_blank">http://www.subrion.org/download/</a>.

Download and save the archive in your project directory using your browser, or command line.

```bash
curl -LO http://tools.subrion.org/get/latest.zip
```

Next, unzip the archive into a subfolder named `subrion` in your project directory.

```bash
unzip latest.zip -d subrion
```

We need to provide write access to a few directories to ensure that all installation checks pass successfully.

```bash
mkdir -p subrion/{tmp,modules,uploads,includes,backup}
chmod -R 775 subrion/{uploads,modules,includes,backup}
chmod 1777 subrion/tmp
docker run -v $PWD/subrion:/app bitnami/apache chown -R :daemon /app/
```

> *The last command updates the group ownership of the application directory to the `daemon` group. This ensures that the daemon processes running inside the containers can access the subrion application.*

### Step 2: Create a VirtualHost

Create a folder name `apache-vhost` in your project directory.

```bash
mkdir apache-vhost
```

Create a file named `subrion.conf` in the `apache-vhost` directory with the following contents.

```apache
LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so
<VirtualHost *:80>
  ServerName subrion.example.com
  DocumentRoot "/app"
  ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://php-fpm:9000/app/$1
  <Directory "/app">
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
  </Directory>
</VirtualHost>
```

In this configuration, the `ProxyPassMatch` directive ensures that all php script parsing tasks are delegated to the PHP-FPM container using the FastCGI interface. The `ServerName` directive specifies the hostname for our Subrion application. The `LoadModule` directive enables the `mod_proxy_fcgi` Apache extension needed by Subrion.

Notice that the hostname we've used for connecting to PHP-FPM is `php-fpm`. We will use this as the alias when linking the PHP-FPM container to the Apache container in the Docker Compose definition.

# Using Docker Compose

The easiest way to get up and running is using
<a href="https://docs.docker.com/compose/" target="_blank">Docker Compose</a>. It uses one YAML file
to define the different containers your application will use, the way they are configured as well
as the links between different containers.

### Step 1: Install Docker Compose

Follow this guide for installing Docker Compose.

- <a href="https://docs.docker.com/compose/install/" target="_blank">https://docs.docker.com/compose/install/</a>

### Step 2: Copy Docker Compose definition

Copy the definition below and save it as `docker-compose.yml` in your project directory.

The following `docker-compose.yml` file will be used to orchestrate the launch of the MariaDB, PHP-FPM and Apache containers using docker-compose.

```yaml
version: '2'

services:
  mariadb:
    image: bitnami/mariadb:10.1.25-r0
    environment:
      - MARIADB_ROOT_PASSWORD=root-password
      - MARIADB_USER=subrion
      - MARIADB_PASSWORD=my-password
      - MARIADB_DATABASE=subriondb
    volumes:
      - ./mariadb-data:/bitnami/mariadb

  php-fpm:
    image: bitnami/php-fpm:5.6.31-r0
    depends_on:
      - mariadb
    volumes:
      - ./subrion:/app

  apache:
    image: bitnami/apache:2.4.27-r0
    ports:
      - 80:80
    depends_on:
      - subrion
    volumes:
      - ./apache-vhost:/bitnami/apache/conf/vhosts
      - ./subrion:/app
```

In the docker compose definition we specified the `MARIADB_USER`, `MARIADB_PASSWORD` and `MARIADB_DATABASE` parameters in the environment of the MariaDB container. The MariaDB container uses these parameters to setup a user and database on the first run. The same credentials should be used to complete the database connection setup in Subrion. The volume mounted at the `/bitnami/mariadb` path of the container ensures persistence of the MariaDB data.

We use the `volumes` property to mount the Subrion application source in the PHP-FPM container at the `/app` path of the container. The link to the MariaDB container allows the PHP-FPM container to access the database server using the `mariadb` hostname.

With the help of docker links, the Apache container will be able to address the PHP-FPM container using the `php-fpm` hostname. The `subrion:/app` volume in the Apache container ensures that the Subrion application source is accessible to the Apache daemon allowing it to be able to serve Subrion's static assets.

Finally, we expose the Apache container port 80 on the host port 80, making the Subrion application accessible over the network.

### Step 3: Running Docker Compose

It's really easy to start a Docker Compose app.

```bash
docker-compose up
```

Docker Compose will show log output from all the containers in your application.

### Step 4: Finishing the setup

To access the `ServerName` we set up in our VirtualHost configuration, you may need to create an entry in your `/etc/hosts` file to point `subrion.example.com` to `localhost`.

```bash
echo '127.0.0.1 subrion.example.com' | sudo tee -a /etc/hosts
```

Navigate to <a href="http://subrion.example.com/install" target="_blank">http://subrion.example.com/install</a> in your browser to complete the installation of Subrion.

In the database connection settings set the hostname to `mariadb`, username to `subrion`, password to `my-password`, database name to `subriondb` and complete the installation.

# Backing up Subrion

The Subrion installation and its current state can easily be backed up by backing up the contents of the `docker-subrion` project directory. Follow these simple instructions to generate a backup.

First stop any running instances of the Subrion application stack to ensure that the application is not updated while the backup is being generated.

```bash
docker-compose stop
```

We can now generate a timestamped bzipped tarball of the `docker-subrion` directory using the following commands.

```bash
pushd .
cd ../
sudo tar -jcpf docker-subrion-$(date +%s).tar.bz2 docker-subrion
```

Now that we have successfully backed up our Subrion installation, we can start up Subrion again.

```bash
popd
docker-compose start
```

# Restoring a Backup

The Subrion application backup can easily be restored by simply extracting the contents of the tarball containing the backup and starting the application stack using `docker-compose` from inside the extracted `docker-subrion` directory.

Before restoring a backup we need to stop and remove any existing containers to make sure new containers are created when we start up Subrion again.

```bash
docker-compose stop
docker-compose rm -v
```

Now we can go ahead and extract the contents of the backup tarball.

> **NOTE:** Upon extraction the `docker-subrion` directory will be created, so please ensure that this directory does not already exist at the location of extraction.

```bash
sudo tar -xf docker-subrion-<BACKUP_TIMESTAMP>.tar.bz2 --same-owner
```

*Replace `BACKUP_TIMESTAMP` in the above command with the timestamp of the backup you wish to restore*

Now its time to start up Subrion

```bash
cd docker-subrion/
docker-compose up
```
