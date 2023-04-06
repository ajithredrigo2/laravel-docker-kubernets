# Deploying a Laravel Application using Docker & Kubernets

To containerize an application refers to the process of adapting an application and its components in order to be able to run it in lightweight environments known as containers. Such environments are isolated and disposable, and can be leveraged for developing, testing, and deploying applications to production.

In this guide, we’ll use Docker Compose to containerize a Laravel application for development. When you’re finished, you’ll have a demo Laravel application running on three separate service containers:

An app service running PHP7.4-apache;
A db service running MySQL 8.0;
An apache service that uses the app service to parse PHP code before serving the Laravel application to the final user.
To allow for a streamlined development process and facilitate application debugging, we’ll keep application files in sync by using shared volumes. We’ll also see how to use docker-compose exec commands to run Composer and Artisan on the app container.

# Prerequisites
### Install Docker and Docker Compose: 
    You need to have Docker and Docker Compose installed on your local machine. Docker is used to build and run the Docker images, and Docker Compose is used to manage the containers.
    https://docs.docker.com/compose/install/linux/#install-the-plugin-manually

### Kubernetes cluster: 
    You need to have access to a Kubernetes cluster to deploy the application. You can use a managed Kubernetes service like Google Kubernetes Engine (GKE), Amazon Elastic Kubernetes Service (EKS), or Microsoft Azure Kubernetes Service (AKS), or you can set up your own Kubernetes cluster using tools like kops, kubeadm, or Rancher.

### Install kubectl: 
    You need to have the kubectl command-line tool installed on your local machine to interact with the Kubernetes cluster.

### Docker registry: 
    You need to have a Docker registry to store the Docker images. You can use a public registry like Docker Hub, or you can set up your own private registry using tools like Harbor or Nexus.

### Laravel application: 
    You need to have a Laravel application that you want to deploy. The application should be containerized using Docker, and the Docker image should be stored in the Docker registry.

### Kubernetes manifests: 
    You need to have Kubernetes manifests that describe how to deploy the Laravel application. The manifests should include a deployment, a service, and any other required resources like secrets or config maps.

### Environment variables: 
    You need to define any environment variables required by the Laravel application, like database credentials, API keys, or other configuration values. These can be defined in the Kubernetes manifests or using a separate configuration tool like Kubernetes ConfigMaps or Secrets.

# Step 1 — Obtaining the Demo Application
To get started, we’ll fetch the demo Laravel application from its Github repository. We’re interested in the tutorial-01 branch, which contains the basic Laravel application we’ve created in the first guide of this series.

To obtain the application code that is compatible with this tutorial, download release tutorial-1.0.1 to your home directory with:

```
cd ~
curl -L https://github.com/ajithredrigo2/laravel-docker-kubernets/archive/test.zip -o laravel.zip
```
We’ll need the unzip command to unpack the application code. In case you haven’t installed this package before, do so now with:

```
sudo apt update
sudo apt install unzip
```

Now, unzip the contents of the application and rename the unpacked directory for easier access:
```
unzip laravel.zip
sudo mv laravel-docker-kubernets-test laravel-docker-kubernets
```
Navigate to the laravel directory:
```
cd laravel-docker-kubernets
```
In the next step, we’ll create a .env configuration file to set up the application.

# Step 2 — Setting Up the Application’s .env File
The Laravel configuration files are located in a directory called config, inside the application’s root directory. Additionally, a .env file is used to set up environment-dependent configuration, such as credentials and any information that might vary between deploys. This file is not included in revision control.

Warning: The environment configuration file contains sensitive information about your server, including database credentials and security keys. For that reason, you should never share this file publicly.

The values contained in the .env file will take precedence over the values set in regular configuration files located at the config directory. Each installation on a new environment requires a tailored environment file to define things such as database connection settings, debug options, application URL, among other items that may vary depending on which environment the application is running.

We’ll now create a new .env file to customize the configuration options for the development environment we’re setting up. Laravel comes with an example.env file that we can copy to create our own:
```
cp .env.example .env
```
Open this file using nano or your text editor of choice:
```
nano .env
```
The current .env file from the laravel application contains settings to use a local MySQL database, with 127.0.0.1 as database host. We need to update the DB_HOST variable so that it points to the database service we will create in our Docker environment. In this guide, we’ll call our database service db. Go ahead and replace the listed value of DB_HOST with the database service name:

### .env
```
APP_NAME=laravel
APP_ENV=dev
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost:8000

LOG_CHANNEL=stack

DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=database_user
DB_PASSWORD=password
...
```
Feel free to also change the database name, username, and password, if you wish. These variables will be leveraged in a later step where we’ll set up the docker-compose.yml file to configure our services.

Save the file when you’re done editing. If you used nano, you can do that by pressing Ctrl+x, then Y and Enter to confirm.

# Step 3 — Setting Up the Application’s Dockerfile
Although both our MySQL and Apache services will be based on default images obtained from the Docker Hub, we still need to build a custom image for the application container. We’ll create a new Dockerfile for that.

Our laravel image will be based on the php:7.4-apache official PHP image from Docker Hub. On top of that basic PHP-FPM environment, we’ll install a few extra PHP modules and the Composer dependency management tool.

We’ll also create a new system user; this is necessary to execute artisan and composer commands while developing the application. The uid setting ensures that the user inside the container has the same uid as your system user on your host machine, where you’re running Docker. This way, any files created by these commands are replicated in the host with the correct permissions. This also means that you’ll be able to use your code editor of choice in the host machine to develop the application that is running inside containers.

Create a new Dockerfile with:
```
nano Dockerfile
```
Copy the following contents to your Dockerfile:

### Dockerfile
```
FROM php:7.4-fpm

# Arguments defined in docker-compose.yml
ARG user
ARG uid

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip

# Clear cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Install PHP extensions
RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd

# Get latest Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Create system user to run Composer and Artisan Commands
RUN useradd -G www-data,root -u $uid -d /home/$user $user
RUN mkdir -p /home/$user/.composer && \
    chown -R $user:$user /home/$user

# Set working directory
WORKDIR /var/www/html/laravel-docker-kubernets

USER $user
```
Don’t forget to save the file when you’re done.

Our Dockerfile starts by defining the base image we’re using: php:7.4-fpm.

After installing system packages and PHP extensions, we install Composer by copying the composer executable from its latest official image to our own application image.

A new system user is then created and set up using the user and uid arguments that were declared at the beginning of the Dockerfile. These values will be injected by Docker Compose at build time.

Finally, we set the default working dir as /var/www and change to the newly created user. This will make sure you’re connecting as a regular user, and that you’re on the right directory, when running composer and artisan commands on the application container.

# Step 4 — Setting Up Apache Configuration and Database Dump Files
When creating development environments with Docker Compose, it is often necessary to share configuration or initialization files with service containers, in order to set up or bootstrap those services. This practice facilitates making changes to configuration files to fine-tune your environment while you’re developing the application.

We’ll now set up a folder with files that will be used to configure and initialize our service containers.

To set up Apache, we’ll share a laravel.conf file that will configure how the application is served. Create the docker-compose/apache folder with:
```
mkdir -p docker-compose/apache
```
Open a new file named laravel.conf within that directory:
```
nano docker-compose/apache/laravel.conf
```
Copy the following Apache configuration to that file:

### docker-compose/apache/laravel.conf
```
<VirtualHost *:80>

  DocumentRoot /var/www/html/laravel-docker-kubernets/public/

  <Directory /var/www/html/laravel-docker-kubernets/>
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
  </Directory>

  ErrorLog ${APACHE_LOG_DIR}/error.log
  CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```
This file will configure Apache to listen on port 80 and use index.php as default index page. It will set the document root to /var/www/public, and then configure Apache to use the app service on port 9000 to process *.php files.

Save and close the file when you’re done editing.

To set up the MySQL database, we’ll share a database dump that will be imported when the container is initialized. This is a feature provided by the MySQL 8.0 image we’ll be using on that container.

Create a new folder for your MySQL initialization files inside the docker-compose folder:
```
mkdir docker-compose/mysql
```
Open a new .sql file:
```
nano docker-compose/mysql/init_db.sql
```
The following MySQL dump is based on the database we’ve set up in our Laravel on LEMP guide. It will create a new table named places. Then, it will populate the table with a set of sample places.

Add the following code to the file:

### docker-compose/mysql/db_init.sql
```
DROP TABLE IF EXISTS `places`;

DROP TABLE IF EXISTS `places`;

CREATE TABLE `places` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
  `visited` tinyint(1) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=12 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

INSERT INTO `places` (name, visited) VALUES ('Berlin',0),('Budapest',0),('Cincinnati',1),('Denver',0),('Helsinki',0),('Lisbon',0),('Moscow',1),('Nairobi',0),('Oslo',1),('Rio',0),('Tokyo',0);
```
The places table contains three fields: id, name, and visited. The visited field is a flag used to identify the places that are still to go. Feel free to change the sample places or include new ones. Save and close the file when you’re done.

We’ve finished setting up the application’s Dockerfile and the service configuration files. Next, we’ll set up Docker Compose to use these files when creating our services.

# Step 5 — Creating a Multi-Container Environment with Docker Compose
Docker Compose enables you to create multi-container environments for applications running on Docker. It uses service definitions to build fully customizable environments with multiple containers that can share networks and data volumes. This allows for a seamless integration between application components.

To set up our service definitions, we’ll create a new file called docker-compose.yml. Typically, this file is located at the root of the application folder, and it defines your containerized environment, including the base images you will use to build your containers, and how your services will interact.

We’ll define three different services in our docker-compose.yml file: app, db, and apache.

The app service will build an image called laravel, based on the Dockerfile we’ve previously created. The container defined by this service will run a php-fpm server to parse PHP code and send the results back to the apache service, which will be running on a separate container. The mysql service defines a container running a MySQL 8.0 server. Our services will share a bridge network named laravel.

The application files will be synchronized on both the app and the apache services via bind mounts. Bind mounts are useful in development environments because they allow for a performant two-way sync between host machine and containers.

Create a new docker-compose.yml file at the root of the application folder:
```
nano docker-compose.yml
```
A typical docker-compose.yml file starts with a version definition, followed by a services node, under which all services are defined. Shared networks are usually defined at the bottom of that file.

To get started, copy this boilerplate code into your docker-compose.yml file:

### docker-compose.yml
```
version: "3.7"
services:


networks:
  laravel:
    driver: bridge
```
We’ll now edit the services node to include the app, db and apache services.

The app Service
The app service will set up a container named laravel-app. It builds a new Docker image based on a Dockerfile located in the same path as the docker-compose.yml file. The new image will be saved locally under the name laravel.

Even though the document root being served as the application is located in the apache container, we need the application files somewhere inside the app container as well, so we’re able to execute command line tasks with the Laravel Artisan tool.

Copy the following service definition under your services node, inside the docker-compose.yml file:

### docker-compose.yml
```
  app:
    build:
      args:
        user: ubuntu
        uid: 1000
      context: ./
      dockerfile: Dockerfile
    image: laravel
    container_name: laravel-app
    restart: unless-stopped
    working_dir: /var/www/html/laravel-docker-kubernets
    ports:
      - 8000:80
    volumes:
      - ./:/var/www/html/laravel-docker-kubernets
      - ./docker-compose/apache:/etc/apache2/sites-enabled/
    networks:
      - laravel
```
These settings do the following:

build: This configuration tells Docker Compose to build a local image for the app service, using the specified path (context) and Dockerfile for instructions. The arguments user and uid are injected into the Dockerfile to customize user creation commands at build time.
image: The name that will be used for the image being built.
container_name: Sets up the container name for this service.
restart: Always restart, unless the service is stopped.
working_dir: Sets the default directory for this service as /var/www.
volumes: Creates a shared volume that will synchronize contents from the current directory to /var/www inside the container. Notice that this is not your document root, since that will live in the apache container.
networks: Sets up this service to use a network named laravel.
The db Service
The db service uses a pre-built MySQL 8.0 image from Docker Hub. Because Docker Compose automatically loads .env variable files located in the same directory as the docker-compose.yml file, we can obtain our database settings from the Laravel .env file we created in a previous step.

Include the following service definition in your services node, right after the app service:

### docker-compose.yml
```
  db:
    image: mysql:8.0
    container_name: laravel-db
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${DB_DATABASE}
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_USER: ${DB_USERNAME}
      SERVICE_TAGS: dev
      SERVICE_NAME: mysql
    volumes:
      - ./docker-compose/mysql:/docker-entrypoint-initdb.d
    networks:
      - laravel
```
These settings do the following:

image: Defines the Docker image that should be used for this container. In this case, we’re using a MySQL 8.0 image from Docker Hub.
container_name: Sets up the container name for this service: laravel-db.
restart: Always restart this service, unless it is explicitly stopped.
environment: Defines environment variables in the new container. We’re using values obtained from the Laravel .env file to set up our MySQL service, which will automatically create a new database and user based on the provided environment variables.
volumes: Creates a volume to share a .sql database dump that will be used to initialize the application database. The MySQL image will automatically import .sql files placed in the /docker-entrypoint-initdb.d directory inside the container.
networks: Sets up this service to use a network named laravel.

Finished docker-compose.yml File
This is how our finished docker-compose.yml file looks like:

### docker-compose.yml
```
version: "3.7"
services:
  app:
    build:
      args:
        user: ubuntu
        uid: 1000
      context: ./
      dockerfile: Dockerfile
    image: laravel
    container_name: laravel-app
    restart: unless-stopped
    working_dir: /var/www/html/laravel-docker-kubernets
    ports:
      - 8000:80
    volumes:
      - ./:/var/www/html/laravel-docker-kubernets
      - ./docker-compose/apache:/etc/apache2/sites-enabled/
    networks:
      - laravel

  db:
    image: mysql:8.0
    container_name: laravel-db
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${DB_DATABASE}
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      SERVICE_TAGS: dev
      SERVICE_NAME: mysql
    ports:
      - "3307:3306"
    volumes:
      - ./docker-compose/mysql:/docker-entrypoint-initdb.d
    networks:
     - laravel

networks:
  laravel:
    driver: bridge
```
Make sure you save the file when you’re done.

# Step 6 — Running the Application with Docker Compose
We’ll now use docker-compose commands to build the application image and run the services we specified in our setup.

Build the app image with the following command:
```
docker-compose build app
```
This command might take a few minutes to complete. You’ll see output similar to this:

### Output
```
Building app
Step 1/12 : FROM php:7.4-apache
 ---> 20a3732f422b
Step 2/12 : ARG user
 ---> Using cache
 ---> a7bed5ac403b
Step 3/12 : ARG uid
 ---> Using cache
 ---> 8be869e818d3
Step 4/12 : RUN apt-get update && apt-get install -y     git     curl     libpng-dev     libonig-dev     libxml2-dev     zip     unzip
 ---> Using cache
 ---> aa20d4585048
Step 5/12 : RUN apt-get clean && rm -rf /var/lib/apt/lists/*
 ---> Using cache
 ---> 6921d032419e
Step 6/12 : RUN docker-php-ext-install pdo pdo_mysql mbstring mysqli exif pcntl bcmath gd
 ---> Using cache
 ---> bfcb774b02ee
Step 7/12 : RUN curl --silent --show-error "https://getcomposer.org/installer" | php -- --install-dir=/usr/local/bin --filename=composer
 ---> Using cache
 ---> cb8c993a7f2a
Step 8/12 : COPY --from=composer:1.10.1 /usr/bin/composer /usr/bin/composer
 ---> Using cache
 ---> 3fc07ec2e29b
Step 9/12 : RUN useradd -G www-data,root -u $uid -d /home/$user $user
 ---> Using cache
 ---> f0f7019fe60b
Step 10/12 : RUN mkdir -p /home/$user/.composer &&     chown -R $user:$user /home/$user
 ---> Using cache
 ---> eec0846b2473
Step 11/12 : WORKDIR /var/www/html/laravel-docker-kubernets
 ---> Using cache
 ---> bda202b3577b
Step 12/12 : USER $user
 ---> Using cache
 ---> 6ea7767cc5bf

Successfully built 6ea7767cc5bf
Successfully tagged laravel:latest
```
When the build is finished, you can run the environment in background mode with:
```
docker-compose up -d
```
### Output
```
Creating laravel-db    ... done
Creating laravel-app   ... done
```
This will run your containers in the background. To show information about the state of your active services, run:
```
docker-compose ps
```
You’ll see output like this:

### Output
```
   Name                  Command               State                          Ports
----------------------------------------------------------------------------------------------------------
laravel-app   docker-php-entrypoint apac ...   Up      0.0.0.0:8000->80/tcp,:::8000->80/tcp
laravel-db    docker-entrypoint.sh mysqld      Up      0.0.0.0:3307->3306/tcp,:::3307->3306/tcp, 33060/tcp
```
Your environment is now up and running, but we still need to execute a couple commands to finish setting up the application. You can use the docker-compose exec command to execute commands in the service containers, such as an ls -l to show detailed information about files in the application directory:
```
docker-compose exec app ls -l
```
### Output
```
total 364
-rw-r--r--  1 ubuntu   ubuntu    901 Mar  7 09:35 Dockerfile
-rw-rw-r--  1 ubuntu   ubuntu   3958 Apr 12  2022 README.md
drwxrwxr-x  7 ubuntu   ubuntu   4096 Apr 12  2022 app
-rwxr-xr-x  1 ubuntu   ubuntu   1686 Apr 12  2022 artisan
drwxrwxr-x  3 ubuntu   ubuntu   4096 Apr 12  2022 bootstrap
-rw-rw-r--  1 ubuntu   ubuntu   1745 Apr 12  2022 composer.json
-rw-rw-r--  1 ubuntu   ubuntu 283765 Mar  7 09:31 composer.lock
drwxrwxr-x  2 ubuntu   ubuntu   4096 Apr 12  2022 config
drwxrwxr-x  5 ubuntu   ubuntu   4096 Apr 12  2022 database
-rw-rw-r--  1 ubuntu   ubuntu   1162 Mar  7 20:40 ddagent-install.log
drwxrwxr-x  4 ubuntu   ubuntu   4096 Mar  7 08:29 docker-compose
-rw-r--r--  1 ubuntu   ubuntu    955 Mar  7 09:56 docker-compose.yml
-rw-rw-r--  1 ubuntu   ubuntu    473 Apr 12  2022 package.json
-rw-rw-r--  1 ubuntu   ubuntu   1202 Apr 12  2022 phpunit.xml
drwxrwxr-x  2 ubuntu   ubuntu   4096 Mar  7 09:52 public
drwxrwxr-x  6 ubuntu   ubuntu   4096 Apr 12  2022 resources
drwxrwxr-x  2 ubuntu   ubuntu   4096 Apr 12  2022 routes
-rw-rw-r--  1 ubuntu   ubuntu    569 Apr 12  2022 server.php
drwxrwxrwx  5 www-data ubuntu   4096 Apr 12  2022 storage
drwxrwxr-x  4 ubuntu   ubuntu   4096 Apr 12  2022 tests
drwxrwxr-x 42 ubuntu   ubuntu   4096 Mar  7 09:32 vendor
-rw-rw-r--  1 ubuntu   ubuntu    559 Apr 12  2022 webpack.mix.js
```
We’ll now run composer install to install the application dependencies:
```
docker-compose exec app rm -rf vendor composer.lock
docker-compose exec app composer install
```
You’ll see output like this:

### Output
```
No composer.lock file present. Updating dependencies to latest instead of installing from lock file. See https://getcomposer.org/install for more information.
. . .
Lock file operations: 89 installs, 0 updates, 0 removals
  - Locking doctrine/inflector (2.0.4)
  - Locking doctrine/instantiator (1.4.1)
  - Locking doctrine/lexer (1.2.3)
  - Locking dragonmantank/cron-expression (v2.3.1)
  - Locking egulias/email-validator (2.1.25)
  - Locking facade/flare-client-php (1.9.1)
  - Locking facade/ignition (1.18.1)
  - Locking facade/ignition-contracts (1.0.2)
  - Locking fideloper/proxy (4.4.1)
  - Locking filp/whoops (2.14.5)
. . .
Writing lock file
Installing dependencies from lock file (including require-dev)
Package operations: 89 installs, 0 updates, 0 removals
  - Downloading doctrine/inflector (2.0.4)
  - Downloading doctrine/lexer (1.2.3)
  - Downloading dragonmantank/cron-expression (v2.3.1)
  - Downloading symfony/polyfill-php80 (v1.25.0)
  - Downloading symfony/polyfill-php72 (v1.25.0)
  - Downloading symfony/polyfill-mbstring (v1.25.0)
  - Downloading symfony/var-dumper (v4.4.39)
  - Downloading symfony/deprecation-contracts (v2.5.1)
. . .
Generating optimized autoload files
> Illuminate\Foundation\ComposerScripts::postAutoloadDump
> @php artisan package:discover --ansi
Discovered Package: facade/ignition
Discovered Package: fideloper/proxy
Discovered Package: laravel/tinker
Discovered Package: nesbot/carbon
Discovered Package: nunomaduro/collision
Package manifest generated successfully.
```
The last thing we need to do before testing the application is to generate a unique application key with the artisan Laravel command-line tool. This key is used to encrypt user sessions and other sensitive data:
```
docker-compose exec app php artisan key:generate

```
### Output
```
Application key set successfully.
```
Now go to your browser and access your server’s domain name or IP address on port 8000:
```
http://server_domain_or_IP:8000
```
### Note: In case you are running this demo on your local machine, use http://localhost:8000 to access the application from your browser.

You’ll see a page like this:

Demo Laravel Application

You can use the logs command to check the logs generated by your services:
```
docker-compose logs app
```
```
Attaching to laravel-app
. . .
laravel-app | [Wed Mar 08 15:20:17.800929 2023] [mpm_prefork:notice] [pid 1] AH00163: Apache/2.4.54 (Debian) PHP/7.4.33 configured -- resuming normal operations
laravel-app | [Wed Mar 08 15:20:17.801085 2023] [core:notice] [pid 1] AH00094: Command line: 'apache2 -D FOREGROUND'
laravel-app | 172.26.0.1 - - [08/Mar/2023:16:19:20 +0000] "GET / HTTP/1.1" 200 18745 "-" "curl/7.68.0" Gecko/20100101 Firefox/89.0"
```
If you want to pause your Docker Compose environment while keeping the state of all its services, run:
```
docker-compose pause
```
### Output
```
Pausing laravel-db    ... done
Pausing laravel-app   ... done
```
You can then resume your services with:
```
docker-compose unpause
```
### Output
```
Unpausing laravel-app   ... done
Unpausing laravel-db    ... done
```
To shut down your Docker Compose environment and remove all of its containers, networks, and volumes, run:
```
docker-compose down
```
### Output
```
Stopping laravel-app ... done
Stopping laravel-db  ... done
Removing laravel-app ... done
Removing laravel-db  ... done
Removing network laravel
```
For an overview of all Docker Compose commands, please check the Docker Compose command-line reference.

# Conclusion
In this guide, we’ve set up a Docker environment with three containers using Docker Compose to define our infrastructure in a YAML file.

From this point on, you can work on your Laravel application without needing to install and set up a local web server for development and testing. Moreover, you’ll be working with a disposable environment that can be easily replicated and distributed, which can be helpful while developing your application and also when moving towards a production environment.
