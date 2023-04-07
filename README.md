# Deploying a Laravel Application using Docker & Kubernets

Deploying a Laravel application using Docker and Kubernetes is an efficient and scalable way to manage your application's infrastructure. Docker allows you to create containerized environments for your application, while Kubernetes provides tools for managing and scaling those containers.

To get started, you'll need to create a Docker image of your Laravel application. This can be done by creating a Dockerfile that specifies the dependencies and configuration for your application. Once the Docker image is created, it can be deployed to a Kubernetes cluster.

In Kubernetes, you'll create a deployment object that specifies how many instances of your containerized application should be running at any given time. You can also define a service object that provides a stable IP address and DNS name for your application.

# Prerequisites
### Install Docker and Docker Compose: 
You need to have [Docker](https://docs.docker.com/engine/install/ubuntu/) and [Docker Compose](https://docs.docker.com/compose/install/linux/#install-the-plugin-manually) installed on your local machine. Docker is used to build and run the Docker images, and Docker Compose is used to manage the containers.

### Kubernetes cluster: 
You need to have access to a Kubernetes cluster to deploy the application. You can use a managed Kubernetes service like Google Kubernetes Engine (GKE), Amazon Elastic Kubernetes Service (EKS), or Microsoft Azure Kubernetes Service (AKS), or you can set up your own Kubernetes cluster using tools like kops, kubeadm, or Rancher.

### Install kubectl: 
You need to have the [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) command-line tool installed on your local machine to interact with the Kubernetes cluster.

### Laravel application: 
You need to have a Laravel application that you want to deploy. The application should be containerized using Docker, and the Docker image should be stored in the Docker registry.

### Install Jenkins
you need to install [Jenkins](https://www.jenkins.io/doc/book/installing/linux/), which is a continuous integration (CI) server that supports a wide range of tools and technologies. Adopting a CI process ensures that all developers' working copies of code are regularly merged into a shared trunk. Once a change is committed to the repository, the product is automatically rebuilt and tested.

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
    volumes:
      - ./:/var/www/html/laravel-docker-kubernets
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

  nginx:
    image: nginx:alpine
    container_name: laravel-nginx
    restart: unless-stopped
    ports:
      - "8000:80"
    volumes:
      - ./:/var/www/html/laravel-docker-kubernets
      - ./docker-compose/nginx:/etc/nginx/conf.d
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
Creating laravel-nginx ... done
```

# Step 7 Create a Kubernetes deployment to run Laravel application
This deployment.yaml file creates a Kubernetes deployment with one replica that runs the my-app-image Docker image. The deployment specifies the container port to expose, as well as environment variables for the PostgreSQL database connection details.

### deployment.yml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: laravel-app
  template:
    metadata:
      labels:
        app: laravel-app
    spec:
      containers:
      - name: laravel-app
        image: ajithredrigo2/laravel:laravel
        ports:
        - containerPort: 8000
        env:
        - name: DB_HOST
          value: mysql
        - name: DB_PORT
          value: "3306"
        - name: DB_DATABASE
          value: laravel
        - name: DB_USERNAME
          value: root
        - name: DB_PASSWORD
          value: Ama$@In*a6dP
``` 

# Step 8 Create a Kubernetes service to expose Laravel application
This service.yaml file creates a Kubernetes service of type LoadBalancer that exposes the container port of the laravel-app deployment and assigns an external IP address.
### service.yml
```
apiVersion: v1
kind: Service
metadata:
  name: laravel-service
spec:
  selector:
    app: laravel-app
  ports:
  - name: http
    port: 8000
  type: LoadBalancer
```
# Step 8 Apply Deployment and Service
Apply the Kubernetes deployment and service YAML files using the following command:
```
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```
Now the laravel application deployed to the kubernets cluster and expose it to intenet.

### Step 9 Setup Jenkins Pipeline
After install jenkins, Create a group for jenkins and docker using the following command,
```
sudo usermod -a -G docker jenkins
```
This will used to ecexute docker commands in jenkins shell.

Create permission for jenkins to project folder
```
sudo chown -R jenkins:www-data /var/www/html/laravel-docker-kubernets
```
Create SecretFile access in jenkins to kubernets
```
    1. Download config file from
        cd ~/.kube/
    2. Set permission for client key 
        sudo chown jenkins ~/.minikube/profiles/minikube/client.key
    3. Insert client.key to Jenkins
        Go to Jenkins --> Manage --> Credentials --> GLobal --> Add --> SecretFile
```

Now create pipeline script in Jenkins
```
pipeline {
    agent {
        node {
            label "master"
            customWorkspace "/var/www/html/laravel-docker-kubernets"
        }
    }
    environment {
        KUBECONFIG = credentials('kubeconfig')
        DOCKER_REGISTRY = 'registry.hub.docker.com'
        DOCKER_USERNAME = 'ajithredrigo2'
        // DOCKER_PASSWORD = 'Redrigo@j2548'
        KUBE_NAMESPACE = 'default'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/ajithredrigo2/laravel-docker-kubernets.git', branch: 'main', credentialsId: 'jenkins'
            }
        }
        
        stage('Docker Build') {
            steps {
                sh 'docker-compose build app'
            }
        }
        
        stage('Run tests') {
            steps {
                sh "php vendor/bin/phpunit"
            }
        }
        
        stage('Docker Tag and Push') {
            steps {
                sh 'docker login -u ajithredrigo2 -p $DOCKER_PASSWORD'
                sh 'docker tag laravel ajithredrigo2/laravel:laravel'
                sh 'docker push ajithredrigo2/laravel:laravel'
            }
        }
        
        stage('Docker Up') {
            steps {
                sh "docker-compose up -d"
                sh "docker-compose exec -T app php artisan optimize:clear"
            }
        }
       
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    try {
                        sh "envsubst < deployment.yml | kubectl apply -f -"
                        sh "envsubst < service.yml | kubectl apply -f -"
                        sh "kubectl rollout status deployment laravel-deployment -n ${KUBE_NAMESPACE}"
                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        sh "kubectl rollout undo deployment laravel-deployment -n ${KUBE_NAMESPACE}"
                        throw e
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                if (currentBuild.result == 'FAILURE') {
                    sh "kubectl rollout status deployment laravel-deployment -n ${KUBE_NAMESPACE}"
                    sh "kubectl rollout undo deployment laravel-deployment -n ${KUBE_NAMESPACE}"
                }
            }
        }
    }
}
```
In the Build stage, the build steps are defined. This could include docker compiling source code, generating documentation, or creating a build artifact.

In the Test stage, the tests are defined. This could include unit tests, integration tests, or acceptance tests.

In the Deploy stage, the deployment steps are defined. This could include deploying the build artifact to a staging environment, running database migrations, or configuring load balancers.

In the Post stage, a deployed application revert back to old revision if any error occurs in previous line.

After build the pipeline the output might be
### Output
```
Started by user jenkins
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins in /var/lib/jenkins/workspace/laravel-docker-kubernets
[Pipeline] {
[Pipeline] ws
Running in /var/www/html/laravel-docker-kubernets
[Pipeline] {
[Pipeline] withCredentials
Masking supported pattern matches of $KUBECONFIG
[Pipeline] {
[Pipeline] withEnv
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Checkout)
[Pipeline] git
The recommended git tool is: NONE
using credential jenkins
 > git rev-parse --resolve-git-dir /var/www/html/laravel-docker-kubernets/.git # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/ajithredrigo2/laravel-docker-kubernets.git # timeout=10
Fetching upstream changes from https://github.com/ajithredrigo2/laravel-docker-kubernets.git
 > git --version # timeout=10
 > git --version # 'git version 2.25.1'
using GIT_SSH to set credentials 
Verifying host key using known hosts file
You're using 'Known hosts file' strategy to verify ssh host keys, but your known_hosts file does not exist, please go to 'Manage Jenkins' -> 'Configure Global Security' -> 'Git Host Key Verification Configuration' and configure host key verification.
 > git fetch --tags --force --progress -- https://github.com/ajithredrigo2/laravel-docker-kubernets.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/main^{commit} # timeout=10
Checking out Revision c8a91d17866cb93475ceebd97ede814ba5c2b0ec (refs/remotes/origin/main)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f c8a91d17866cb93475ceebd97ede814ba5c2b0ec # timeout=10
 > git branch -a -v --no-abbrev # timeout=10
 > git branch -D main # timeout=10
 > git checkout -b main c8a91d17866cb93475ceebd97ede814ba5c2b0ec # timeout=10
Commit message: "ingress"
 > git rev-list --no-walk c8a91d17866cb93475ceebd97ede814ba5c2b0ec # timeout=10
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Docker Build)
[Pipeline] sh
+ docker-compose build app
Building app
Step 1/12 : FROM php:7.4-fpm
 ---> 38f2b691dcb8
Step 2/12 : ARG user
 ---> Using cache
 ---> d32d580a198d
Step 3/12 : ARG uid
 ---> Using cache
 ---> 611f9c39229c
Step 4/12 : RUN apt-get update && apt-get install -y     git     curl     libpng-dev     libonig-dev     libxml2-dev     zip     unzip
 ---> Using cache
 ---> a347bacec3d8
Step 5/12 : RUN apt-get clean && rm -rf /var/lib/apt/lists/*
 ---> Using cache
 ---> 5f8278dad545
Step 6/12 : RUN docker-php-ext-install pdo pdo_mysql mbstring mysqli exif pcntl bcmath gd
 ---> Using cache
 ---> e3bdec99d19f
Step 7/12 : RUN curl --silent --show-error "https://getcomposer.org/installer" | php -- --install-dir=/usr/local/bin --filename=composer
 ---> Using cache
 ---> 2e045777e6ce
Step 8/12 : COPY --from=composer:1.10.1 /usr/bin/composer /usr/bin/composer
 ---> Using cache
 ---> 0861b71f47c0
Step 9/12 : RUN useradd -G www-data,root -u $uid -d /home/$user $user
 ---> Using cache
 ---> 2c5dc556beaf
Step 10/12 : RUN mkdir -p /home/$user/.composer &&     chown -R $user:$user /home/$user
 ---> Using cache
 ---> 9b283b1290bc
Step 11/12 : WORKDIR /var/www/html/laravel-docker-kubernets
 ---> Using cache
 ---> 2e66f8cf877b
Step 12/12 : USER $user
 ---> Using cache
 ---> 7bea978cd832

Successfully built 7bea978cd832
Successfully tagged laravel:latest
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Run tests)
[Pipeline] sh
+ php vendor/bin/phpunit
PHPUnit 9.6.6 by Sebastian Bergmann and contributors.

..                                                                  2 / 2 (100%)

Time: 00:00.067, Memory: 20.00 MB

OK (2 tests, 2 assertions)
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Docker Tag and Push)
[Pipeline] sh
+ docker login -u ajithredrigo2 -p Redrigo@j2548
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /var/lib/jenkins/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[Pipeline] sh
+ docker tag laravel ajithredrigo2/laravel:laravel
[Pipeline] sh
+ docker push ajithredrigo2/laravel:laravel
The push refers to repository [docker.io/ajithredrigo2/laravel]
9bf121a508e9: Preparing
00d5038b895e: Preparing
a01648a1f601: Preparing
98711257115c: Preparing
2fc79ae99e03: Preparing
0e87d9b8d675: Preparing
860bd024b846: Preparing
d6ed42ec7c6e: Preparing
5e65a6c61859: Preparing
d78098596d78: Preparing
7c314756ee72: Preparing
89982c6135ad: Preparing
91fd2792fa74: Preparing
08cc615b0242: Preparing
44148371c697: Preparing
797a7c0590e0: Preparing
f60117696410: Preparing
ec4a38999118: Preparing
0e87d9b8d675: Waiting
860bd024b846: Waiting
d6ed42ec7c6e: Waiting
5e65a6c61859: Waiting
d78098596d78: Waiting
7c314756ee72: Waiting
797a7c0590e0: Waiting
89982c6135ad: Waiting
f60117696410: Waiting
91fd2792fa74: Waiting
ec4a38999118: Waiting
08cc615b0242: Waiting
44148371c697: Waiting
9bf121a508e9: Layer already exists
2fc79ae99e03: Layer already exists
a01648a1f601: Layer already exists
00d5038b895e: Layer already exists
98711257115c: Layer already exists
5e65a6c61859: Layer already exists
0e87d9b8d675: Layer already exists
d6ed42ec7c6e: Layer already exists
860bd024b846: Layer already exists
d78098596d78: Layer already exists
08cc615b0242: Layer already exists
7c314756ee72: Layer already exists
89982c6135ad: Layer already exists
91fd2792fa74: Layer already exists
44148371c697: Layer already exists
f60117696410: Layer already exists
797a7c0590e0: Layer already exists
ec4a38999118: Layer already exists
laravel: digest: sha256:cc9f033f82f1eab69b52db41aa85b35fa240d559a113d4545d927c85d48e9006 size: 4081
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Docker Up)
[Pipeline] sh
+ docker-compose up -d
laravel-app is up-to-date
laravel-nginx is up-to-date
laravel-db is up-to-date
[Pipeline] sh
+ docker-compose exec -T app php artisan optimize:clear
Cached events cleared!
Compiled views cleared!
Application cache cleared!
Route cache cleared!
Configuration cache cleared!
Compiled services and packages files removed!
Caches cleared successfully!
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Deploy to Kubernetes)
[Pipeline] script
[Pipeline] {
[Pipeline] sh
+ envsubst
+ kubectl apply -f -
deployment.apps/laravel-deployment configured
[Pipeline] sh
+ envsubst
+ kubectl apply -f -
service/laravel-service unchanged
[Pipeline] sh
+ kubectl rollout status deployment laravel-deployment -n default
Waiting for deployment "laravel-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "laravel-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "laravel-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "laravel-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "laravel-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "laravel-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "laravel-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "laravel-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "laravel-deployment" successfully rolled out
[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Declarative: Post Actions)
[Pipeline] script
[Pipeline] {
[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // withCredentials
[Pipeline] }
[Pipeline] // ws
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```
    
# Conclusion
In conclusion, deploying a Laravel application using Docker and Kubernetes is a powerful and efficient way to manage the deployment of your application in a production environment.

Docker allows you to package your application into a container that includes all its dependencies and configurations, making it easy to deploy and run in any environment. Kubernetes then allows you to manage and orchestrate the deployment of your containers, providing features like scaling, rolling updates, and self-healing.

To deploy a Laravel application using Docker and Kubernetes, you'll need to create a Docker image of your application, configure a Kubernetes deployment, and create a Kubernetes service to expose your application to the internet. You can then use Kubernetes to manage the deployment and scaling of your application as traffic increases or decreases.

Overall, using Docker and Kubernetes to deploy a Laravel application provides a scalable, reliable, and efficient solution that allows you to focus on developing your application rather than managing the infrastructure.
