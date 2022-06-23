FROM nginx:1.22

# Note: the context of this build is the root dir of dev-env

# Copy certs
COPY ssl /etc/ssl

# Copy nginx configuration
COPY docker-reverse-proxy/readup-server.conf /etc/nginx/readup-server.conf
COPY docker-reverse-proxy/nginx.conf /etc/nginx/nginx.conf

EXPOSE 443