FROM ubuntu
MAINTAINER zpang zpang@dataman-inc.com
#install nginx
RUN apt-get update
RUN apt-get install -y nginx
# forward request and error logs to docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log
RUN ln -sf /dev/stderr /var/log/nginx/error.log
#off nginx daemon
RUN echo "daemon off;" >> /etc/nginx/nginx.conf
