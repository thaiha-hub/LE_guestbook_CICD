# This Dockerfile sets up an NGINX server to serve a simple frontend for the guestbook application
FROM registry.access.redhat.com/ubi10/nginx-126:10.0

# Switch to root user to copy configuration files 
USER 0
# Set working directory
WORKDIR /opt/app-root/src

# Copy frontend source code and NGINX configuration to the container
COPY index.html /opt/app-root/src/index.html
COPY nginx.conf /etc/nginx/nginx.conf

# Expose port 8080
EXPOSE 8080

# Command to run NGINX in the foreground; "daemon off" so that the container keeps running
CMD ["nginx", "-g", "daemon off;"]

