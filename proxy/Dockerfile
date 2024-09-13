FROM nginx:alpine
COPY default.conf /etc/nginx/conf.d/default.conf
RUN  mkdir -p /etc/nginx/ssl
COPY /ssl/nginx.crt /etc/nginx/ssl
COPY /ssl/nginx.key /etc/nginx/ssl