#!/bin/bash
FROM ubuntu:18.04
RUN apt update && apt install -y nginx net-tools vim curl
RUN rm -rf etc/nginx/sites-enabled/* /etc/nginx/conf.d/*
RUN mkdir -p /var/www/html/app/yellow && \
    mkdir -p /var/www/html/app/red && \
    echo "<html style=\"background-color: yellow;\"><body><h1>Welcome to Yellow App!</h1></body></html>" >> /var/www/html/app/yellow/index.html && \
    echo "<html style=\"background-color: red;\"><body><h1>Welcome to Red App!</h1></body></html>" >> /var/www/html/app/red/index.html && \
    chown -R www-data:www-data /var/www/html
COPY default.conf /etc/nginx/conf.d/default.conf
#CMD service nginx start
#CMD ["nginx","-g","daemon off;"]