# Introduction

In this tutorial, we will use Bitnami containers to run the <a href="http://www.redmine.org/" target="_blank">Redmine</a>, taking advantage of Docker features such as linking and volumes.

### [MariaDB](https://github.com/bitnami/bitnami-docker-mariadb)

MariaDB will be used as the database server of choice and will be used by Redmine to store its database schema.

### [Ruby](https://github.com/bitnami/bitnami-docker-ruby)

Redmine is a Ruby on Rails application so we will use the Ruby container to provide Ruby support in the application stack.

### [nginx](https://github.com/bitnami/bitnami-docker-nginx)

We will serve our Redmine app through an nginx server which we will make accessible to the host machine. nginx will act as a reverse proxy to our Ruby container that will run the Redmine server.

# Setting up Redmine

### Step 1: Download Redmine

Create a new project directory that will contain our app's content.

```bash
mkdir docker-redmine
cd docker-redmine
```

The latest version of Redmine can be downloaded from <a href="http://www.redmine.org/projects/redmine/wiki/Download" target="_blank">http://www.redmine.org/projects/redmine/wiki/Download</a>.

Download and save the archive in your project directory using your browser, or command line.

```bash
curl -LO http://www.redmine.org/releases/redmine-3.1.0.tar.gz
```

Next, unzip the archive into a subfolder named `redmine` in your project directory.

```bash
mkdir -p redmine
tar -xf redmine-3.1.0.tar.gz --strip=1 -C redmine/
```

### Step 2: Configure Redmine's database credentials

Since we will be using MariaDB as the database server, we need to configure Redmine to connect to the database with the right credentials and connection parameters.

Create the file named `redmine/config/database.yml` with the following configuration:

```yaml
production:
  adapter: mysql2
  database: redminedb
  host: mariadb
  username: redmine
  password: "my-password"
  encoding: utf8
```

Notice that the hostname we have used for our database connection is `mariadb`. We will use this as the alias when setting up the link between the MariaDB and the Ruby containers.

We have also specified a user and database for Redmine. We will specify environment variables to the MariaDB container to create these in the docker compose specification.

### Step 3: Create an nginx Virtual Host

Create a folder name `nginx-vhost` in your project directory.

```bash
mkdir nginx-vhost
```

Create a file named `redmine.conf` in the `nginx-vhost` directory with the following contents:

```nginx
server {
  listen 0.0.0.0:80;
  server_name redmine.example.com;

  location / {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header HOST $http_host;
    proxy_set_header X-NginX-Proxy true;

    proxy_pass http://redmine:3000;
    proxy_redirect off;
  }
}
```

In this configuration, the `proxy_pass` parameter ensures that all Ruby script parsing tasks are delegated to the Ruby container. The `server_name` parameter specifies the hostname for our Redmine application.

Notice that the hostname we've used for connecting to Ruby is `redmine`, we will use this as the alias when linking the Ruby container to the nginx container in the docker compose definition.

# Using Docker Compose

The easiest way to get up and running is using <a href="https://docs.docker.com/compose/" target="_blank">Docker Compose</a>. It uses one YAML file to define the different containers your application will use, the way they are configured as well as the links between different containers.

### Step 1: Install Docker Compose

Follow this guide for installing Docker Compose.

- <a href="https://docs.docker.com/compose/install/" target="_blank">https://docs.docker.com/compose/install/</a>

### Step 2: Docker Compose Specification

Create a file named `docker-compose.yml` in your project directory with the following contents:

```yaml
mariadb:
  image: bitnami/mariadb:10.1.13-r0
  environment:
    - MARIADB_USER=redmine
    - MARIADB_PASSWORD=my-password
    - MARIADB_DATABASE=redminedb
  volumes:
    - ./mariadb-data:/bitnami/mariadb

redmine:
  image: bitnami/ruby
  links:
    - mariadb:mariadb
  volumes:
    - ./redmine:/app
  command: >
    sh -c 'bundle install --without development test \
        && bundle exec rake db:migrate RAILS_ENV=production \
        && bundle exec rake generate_secret_token RAILS_ENV=production \
        && bundle exec rails server -b 0.0.0.0 -p 3000 -e production'

nginx:
  image: bitnami/nginx
  links:
    - redmine:redmine
  ports:
    - 80:80
  volumes:
    - ./nginx-vhost:/bitnami/nginx/conf/vhosts
```

The `docker-compose.yml` file will be used to orchestrate the launch of the MariaDB, Ruby and nginx containers using docker-compose.

In the docker compose definition we specified the `MARIADB_USER`, `MARIADB_PASSWORD` and `MARIADB_DATABASE` parameters in the environment of the MariaDB container. The MariaDB container uses these parameters to setup a user and database on the first run. We have setup these variables according to the `database.yml` configuration above. The volume mounted at the `/bitnami/mariadb` path of the container ensures persistence of the MariaDB data.

We use the `volumes` property to mount the Redmine application source in the Ruby container at the `/app` path of the container. The link to the MariaDB container allows the Ruby container to access the database server using the `mariadb` hostname.

With the help of docker links, the nginx container will be able to address the Ruby container using the `redmine` hostname.

Finally, we expose the nginx container port 80 on the host port 80, making the Redmine application accessible over the network.

### Step 3: Running Docker Compose

It's really easy to start a Docker Compose app.

```bash
docker-compose up
```

Docker Compose will show log output from all the containers in your application.

### Step 4: Access your Redmine instance

To access the `server_name` we set up in our Virtual Host configuration, you may need to create an entry in your `/etc/hosts` file to point `redmine.example.com` to `localhost`.

```bash
echo '127.0.0.1 redmine.example.com' | sudo tee -a /etc/hosts
```

Navigate to <a href="http://redmine.example.com" target="_blank">http://redmine.example.com</a> in your browser and login using the default username and password:

* username: **admin**
* password: **admin**

*Make sure you visit the `Administration` link and `Load the default configuration` before creating any projects.*

# Backing up Redmine

The Redmine installation and its current state can easily be backed up by backing up the contents of the `docker-redmine` project directory. Follow these simple instructions to generate a backup.

First stop any running instances of the Redmine application stack to ensure that the application is not updated while the backup is being generated.

```bash
docker-compose stop
```

We can now generate a timestamped bzipped tarball of the `docker-redmine` directory using the following commands.

```bash
pushd .
cd ../
sudo tar -jcpf docker-redmine-$(date +%s).tar.bz2 docker-redmine
```

Now that we have successfully backed up our Redmine installation, we can start up Redmine again.

```bash
popd
docker-compose start
```

# Restoring a Backup

The Redmine application backup can easily be restored by simply extracting the contents of the tarball containing the backup and starting the application stack using `docker-compose` from inside the extracted `docker-redmine` directory.

Before restoring a backup we need to stop and remove any existing containers to make sure new containers are created when we start up Redmine again.

```bash
docker-compose stop
docker-compose rm -v
```

Now we can go ahead and extract the contents of the backup tarball.

> **NOTE:** Upon extraction the `docker-redmine` directory will be created, so please ensure that this directory does not already exist at the location of extraction.

```bash
sudo tar -xf docker-redmine-<BACKUP_TIMESTAMP>.tar.bz2 --same-owner
```

*Replace `BACKUP_TIMESTAMP` in the above command with the timestamp of the backup you wish to restore*

Now its time to start up Redmine

```bash
cd docker-redmine/
docker-compose up
```
