**Why Docker?** Docker, a powerful tool for creating, deploying, and running applications by using containers, offers a solution. There are numerous methods to create a PHP development environment, opting packages like XAMPP, WampServer or even install PHP, MySQL, and web server software manually. Using Docker with PHP offers numerous benefits, especially in terms of development efficiency, environment consistency, scalability, enhancing security, development workflow, configure management, tooling support etc. This blog post will meticulously guide you through creating a Docker image with PHP 7.4, enabling you to leverage the benefits of containerization for your PHP projects.

Make sure Docker installed & active on your machine
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
├── .docker/
│   ├── nginx/
│   │   └── default.conf
│   └── php/
│       ├── dockerfile
│       └── php.ini
├── docker-compose.yml
└── public
    └── index.php

```

### 1. Setting Up Nginx Configuration (.docker/nginx/default.conf):
Create the <code>default.conf</code> file within the <code>.docker/nginx</code> directory. Paste the following configuration, which serves as a foundation for proxying requests to your PHP application:
```
server {
  # Defines the port on which Nginx listens for incoming traffic
  listen 8082;
  # Adjust if the application's entry point is elsewhere
  root /var/www/html/public/;  
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

### 2. Building the PHP Dockerfile (.docker/php/dockerfile):
Create the <code>dockerfile</code> file within the <code>.docker/php</code> directory. Paste the following content, which defines the instructions for building your PHP container:   

```
FROM php:7.4-fpm

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

### 3. Create php.ini (.docker/php/php.ini):
Create the <code>php.ini</code> file within the <code>.docker/php</code> directory.
```
# Defines the maximum size allowed for uploaded files
upload_max_filesize = 100M
# Specifies the maximum size allowed for the entire HTTP POST request
post_max_size = 108M
```
### 4: Define docker-compose.yml
Create a file named <code>docker-compose.yaml</code> (Docker supports configuration files in YAML format) in your workspace. This docker-compose.yml file defines a two-container setup.
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
Edit docker-compose.yaml file to add the following instructions:
```
#...
services:
  webserver:
    image: nginx:1.21.6-alpine #Docker image to use for the web server (nginx:1.21.6-alpine known for being lightweight)
    container_name: webserver # A custom name for the container
    restart: unless-stopped # Ensures the container restarts automatically if it crashes or exits unexpectedly
    ports:
      - "8082:8082" # Maps port 8082 on your host machine to port 8082 inside the container
    volumes:
      - ./:/var/www/html #Mounts your current directory (./) from the host machine to the /var/www/html directory within the Nginx container.
      - .docker/nginx/:/etc/nginx/conf.d/ # Mounts the .docker/nginx directory from your host machine to the /etc/nginx/conf.d/ directory within the container. 
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
      context: .docker/php # Defines the instructions for building the PHP container image
      dockerfile: dockerfile # Specifies the name of the Dockerfile to use within the context directory
    container_name: backend # A custom name for the container
    volumes:
      - ./:/var/www/html
      - .docker/php/php.ini:/usr/local/etc/php/conf.d/local.ini # Mounts the .docker/php/php.ini file from your host machine to the /usr/local/etc/php/conf.d/local.ini location within the container
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
      - "8082:8082"
    volumes:
      - ./:/var/www/html
      - .docker/nginx/:/etc/nginx/conf.d/
    # command: sh -c "composer install --ignore-platform-reqs && php artisan key:generate"
    networks:
      app-networks:

  backend:
    build:
      context: .docker/php
      dockerfile: dockerfile
    container_name: backend
    volumes:
      - ./:/var/www/html
      - .docker/php/php.ini:/usr/local/etc/php/conf.d/local.ini
    networks:
      app-networks:
```
This docker-compose.yml file sets up two services: webserver (Nginx) and backend (PHP-FPM). It maps ports and directories, and sets up a network for communication between containers.

### 5. Create index.php & Test Docker Setup
Create a simple index.php in the public/ directory:
```
<?php phpinfo();
```

### 6. Run Docker Compose
Navigate to the root of your project where docker-compose.yml is located and run
```
docker-compose up -d
```
Now, reloading http://127.0.0.1:8082/ in your browser should display the PHP info page, indicating that your PHP environment is correctly set up and running.
Now we have a fully functional Docker environment for running PHP 7.4 applications using Nginx and PHP-FPM. This setup is suitable for development and can be extended for production too.
