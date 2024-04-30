# The Developer's Dream: Achieve Reproducible PHP Development with Docker

**Why Docker?** Docker, a powerful tool for creating, deploying, and running applications by using containers, offers a solution. There are numerous methods to create a PHP development environment, opting packages like XAMPP, WampServer or even install PHP, MySQL, and web server software manually. Using Docker with PHP offers numerous benefits, especially in terms of development efficiency, environment consistency, scalability, enhancing security, development workflow, configure management, tooling support etc. This blog post will meticulously guide you through creating a Docker image with PHP 7.4, enabling you to leverage the benefits of containerization for your PHP projects.

Make sure Docker installed & active on your machine. If you are using linux, check docker active status
```
sudo systemctl status docker
```

## Organize Project Structure

Firstly, create a directory for your project
```
mkdir docker-php
cd docker-php
```
Inside docker-php, create the following subdirectories and files:

```
docker-php/
├── src/
│   ├── app1/
│   │   └── index.php
│   └── app2/
│       └── index.php
└── docker/
    ├── docker-compose.yml
    └── services/
        ├── nginx/
        │   └── default.conf
        └── php/
            ├── dockerfile
            └── php.ini

```

### 1. Setting Up Nginx Configuration (docker-php/docker/services/nginx/default.conf):
Create the <code>default.conf</code> file within the <code>docker-php/docker/services/nginx/</code> directory. Paste the following configuration, which serves as a foundation for proxying requests to your PHP application:
```
server {
    # Defines the port on which Nginx listens for incoming traffic
    listen 8080;
    # Adjust if the application's entry point is elsewhere
    root /var/www/html/app1/;
    index index.php index.html;
    error_log /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;

    # Handles requests for PHP files, proxying them to the PHP-FPM backend container. 
    location ~ \.php$ {
        try_files $uri = 404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass backend:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
        gzip_static on;
    }
}

server {
    # Defines the port on which Nginx listens for incoming traffic
    listen 8081;
    # Adjust if the application's entry point is elsewhere
    root /var/www/html/app2/;
    index index.php index.html;
    error_log /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;

    # Handles requests for PHP files, proxying them to the PHP-FPM backend container. 
    location ~ \.php$ {
        try_files $uri = 404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass backend:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
        gzip_static on;
    }
}
```

### 2. Building the PHP Dockerfile (docker-php/docker/services/php/dockerfile):
Create the <code>dockerfile</code> file within the <code>docker-php/docker/services/php/</code> directory. Paste the following content, which defines the instructions for building your PHP container. Here, we'll use the php:8.1.0-fpm image. You can explore other versions on the official Docker Hub page for PHP: https://hub.docker.com/_/php  

```
FROM php:8.1.0-fpm

ENV USER=www
ENV GROUP=www

# Installing System Dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip \
    vim

# Clear Cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Install PHP Dependencies
RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd

# Install Postgre PDO
RUN apt-get update && apt-get install -y libpq-dev && docker-php-ext-install pdo pdo_pgsql

# Get latest Composer
COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer

# Setup Working Directory
WORKDIR /var/www/html

# Create User & GROUP
RUN groupadd -g 1000 ${GROUP} && useradd -u 1000 -ms /bin/bash -g ${GROUP} ${USER}

# Grant Permissions
RUN chown -R ${USER} /var/www/html

# Select User
USER ${USER}

# Copy permission to selected user ...
COPY --chown=${USER}:${GROUP} . .

EXPOSE 9000

CMD ["php-fpm"]
```
This Dockerfile installs necessary libraries, PHP extensions, and Composer.

### 3. Create php.ini (docker-php/docker/services/php/php.ini):
Create the <code>php.ini</code> file within the <code>docker-php/docker/services/php/</code> directory.
```
# Defines the maximum size allowed for uploaded files
upload_max_filesize = 100M
# Specifies the maximum size allowed for the entire HTTP POST request
post_max_size = 108M
```
### 4: Define docker-compose.yml (docker-php/docker/docker-compose.yml)
Create a file named <code>docker-compose.yaml</code> (Docker supports configuration files in YAML format) within the <code>docker-php/docker/</code> directory in your workspace. This docker-compose.yml file defines a two-container setup.
 - Nginx container: Runs the Nginx web server, configured to serve your application's files from the mounted volume and proxying PHP requests to the backend container.
 - PHP container: Built from the provided Dockerfile, it runs the PHP-FPM process to handle PHP requests sent by Nginx.

Initial docker-compose.yml be like
```
version: "3.8"

networks:
  app-networks:

services:
  webserver:

  backend:
```
### Setting Up the Nginx Web Server
Edit docker-compose.yaml file to add the following instructions: (We'll use nginx:1.21.6-alpine image here. Explore other versions on Docker Hub Nginx: https://hub.docker.com/_/nginx)
```
#...
services:
  webserver:
    image: nginx:1.21.6-alpine #Docker image to use for the web server (nginx:1.21.6-alpine known for being lightweight)
    container_name: webserver # A custom name for the container
    restart: unless-stopped # Ensures the container restarts automatically if it crashes or exits unexpectedly
    ports:
      - "8080:8080" # Maps port 8080 on your host machine to port 8080 inside the container
      - "8081:8081" # Maps port 8081 on your host machine to port 8081 inside the container
    volumes:
      - ./../src/app1:/var/www/html/app1 #Mounts your app1 directory (./../src/app1) from the host machine to the /var/www/html/app1 directory within the Nginx container.
      - ./../src/app2:/var/www/html/app2 #Mounts your app2 directory (./../src/app2) from the host machine to the /var/www/html/app2 directory within the Nginx container.
      - ./services/nginx/:/etc/nginx/conf.d/ # Mounts the ./services/nginx/ directory from your host machine to the /etc/nginx/conf.d/ directory within the container. 
    networks:
      app-networks:
```

### Setting Up the PHP Container
Edit docker-compose.yaml file to add the following instructions:

```
#...
services:
  backend:
    build:
      context: services/php # Defines the instructions for building the PHP container image
      dockerfile: dockerfile # Specifies the name of the Dockerfile to use within the context directory
    container_name: backend # A custom name for the container
    volumes:
      - ./../src/app1:/var/www/html/app1
      - ./../src/app2:/var/www/html/app2
      - ./services/php/php.ini:/usr/local/etc/php/conf.d/local.ini # Mounts the ./services/php/php.ini file from your host machine to the /usr/local/etc/php/conf.d/local.ini location within the container
    networks:
      app-networks:
```

### Finalize docker-compose.yml 
After configuring the services, final docker-compose.yml would be like this
```
version: "3.8"

networks:
  app-networks:

services:
  webserver:
    image: nginx:1.21.6-alpine
    container_name: webserver
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "8081:8081"
    volumes:
      - ./../src/app1:/var/www/html/app1
      - ./../src/app2:/var/www/html/app2
      - ./services/nginx/:/etc/nginx/conf.d/
    networks:
      app-networks:

  backend:
    build:
      context: services/php
      dockerfile: dockerfile
    container_name: backend
    volumes:
      - ./../src/app1:/var/www/html/app1
      - ./../src/app2:/var/www/html/app2
      - ./services/php/php.ini:/usr/local/etc/php/conf.d/local.ini
    networks:
      app-networks:
```
This docker-compose.yml file sets up two services: webserver (Nginx) and backend (PHP-FPM). It maps ports and directories, and sets up a network for communication between containers.

### 5. Create index.php & Test Docker Setup
Create a simple index.php in the docker-app/src/app1/ directory:
```
<?php phpinfo();
```
Create another index.php in the docker-app/src/app2/ directory:
```
<?php

echo "This is application service 2";
```

### 6. Run Docker Compose
Navigate to where docker-compose.yml is located and run
```
docker-compose up -d
```
Now, using http://127.0.0.1:8080/ in your browser should display the PHP info page, indicating that your PHP environment is correctly set up and running. And http://127.0.0.1:8081/ will  display <code>This is application service 2</code> 
message.
Now we have a fully functional Docker environment for running php 8.1 applications using Nginx and PHP-FPM. This setup is suitable for development and can be extended for production too. Happy coding.

