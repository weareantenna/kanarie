# Kanarie deploy server

Kanarie can be used to host multiple websites on a single host. It uses [an nginx proxy](https://github.com/jwilder/nginx-proxy) with SSL certificates through [Let's Encrypt](https://letsencrypt.org/).  

It is currently running on [antenna.dev](https://antenna.dev) and this project still includes that landing website.

## Setting up Kanarie

1. Install Docker and Docker compose on the host
2. Create the docker network for all kanarie containers: `docker network create kanarie`
3. Deploy [docker-compose.yml](docker-compose.yml) to the server.
4. Pull and run the containers, `docker-compose pull && docker-compose up -d`

These last two steps should be run through a CI/CD system, like Buddy, so updates can be rolled out efficiently, without having to log in to the server.

## Setting up a project

### Dockerfile
Kanarie just runs a bunch of Docker containers, so you'll need Docker images to run your own. As we won't be mounting any volumes, these containers need to be self sufficient.

Place these files in the Git repository of your project, so they're easy to follow up on. We recommend putting them in a separate `docker/<project> directory.

For static websites, we recommend starting from the `nginx` image.

Example:
```
# docker/<project-name>/Dockerfile
FROM nginx:1.17-alpine

COPY ./<your-directory-with-files-here> /code/public  
COPY ./docker/kanarie/default.conf /etc/nginx/conf.d/default.conf
``` 
For nginx containers, you also want to replace the default.conf file with your own, for static websites, it can look a little something like this:
```
# docker/<project-name>/default.conf
server {
    listen              80;
    server_name         localhost _;
    index               index.html;
    root                /code/public;

    location / {
    }
}

```

For PHP projects, you can use separate php and nginx containers, but for our use cases, it is often enough to just use a single `php` image with apache.

Example:
```
# .docker/php/Dockerfile
FROM php:7.4-apache

ENV APACHE_DOCUMENT_ROOT /var/www/html/public

RUN docker-php-ext-enable pdo_mysql apcu

# Change the default /var/www/html/ apache directory
RUN a2enmod rewrite \
    && sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf \
    && sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf

# Copy all files required to run the project, use a .dockerignore file to skip certain files
COPY .docker/php/php.ini $PHP_INI_DIR/conf.d/php.ini
COPY . /var/www/html/

WORKDIR /var/www/html/
```

Note that these Dockerfiles could also be used to run local dev environments!

### docker-compose file

To deploy the project on a Kanarie host, we'll use a specific docker-compose.yml file.

```yaml
# docker/prod/docker-compose.yml
version: '3.7'

services:
    some-project:
        image: <name-of-your-image-on-a-registry>
        environment:
            - VIRTUAL_HOST=<domainname-of-project>
            - LETSENCRYPT_HOST=<domainname-of-project>

    project-redis:
        image: redis:5.0

networks:
    default:
        external:
            name: kanarie

```

Notes:
- **Important**: The containers of your project must use the same network as the kanarie docker-compose.yml file, otherwise, the nginx-proxy won't be able to connect properly.
- **Important**: The domain name you specify in the environment variables needs to have proper DNS records pointing to the Kanarie host.
- You can also add other containers required here, like a Redis instance.
- By putting the compose file in a separate env directory, you could easily create a staging and production environment with different (sub)domains. 

## Deploying to Kanarie

These steps should be run inside a CI/CD pipeline

1. (Optional) Build the required files for the project, like `npm install` or `composer install`. 
1. (Optional) Make sure there is a valid .env file for the required environment that can be copied into the container.
1. Build the Docker image with the Dockerfile
2. Push the created image to your registry of choice
3. Copy the docker-compose.yml file for the required environment to the Kanarie host
4. Update the containers by running `docker-compose pull && docker-compose up -d`

That's it. Your website should now be accessible on the configured domain.
