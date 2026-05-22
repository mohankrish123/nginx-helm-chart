FROM ubuntu:22.04 AS build

RUN apt-get update && \
    apt-get install -y --no-install-recommends nginx && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN rm -rf /etc/nginx/sites-enabled/* /etc/nginx/conf.d/*

RUN mkdir -p /var/www/html/app/yellow /var/www/html/app/red && \
    echo '<html style="background-color: yellow;"><body><h1>Welcome to Yellow App!</h1></body></html>' > /var/www/html/app/yellow/index.html && \
    echo '<html style="background-color: red;"><body><h1>Welcome to Red App!</h1></body></html>' > /var/www/html/app/red/index.html && \
    chown -R www-data:www-data /var/www/html

COPY default.conf /etc/nginx/conf.d/default.conf

FROM ubuntu:22.04

RUN apt-get update && \
    apt-get install -y --no-install-recommends nginx && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

COPY --from=build /var/www/html /var/www/html
COPY --from=build /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf

RUN rm -rf /etc/nginx/sites-enabled/*

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
