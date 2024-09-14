# Three-Tier Application with Docker

This project implements a three-tier application using Docker and Docker Compose. It consists of a backend written in Go, a MySQL database, and an Nginx proxy. Each service is deployed in its own Docker container and connected via Docker networks. The application demonstrates how to securely serve a Go-based backend through Nginx with HTTPS, and how to persist data using a MySQL database.

## Architecture

1. **Backend**: A Go application that connects to a MySQL database.
2. **Database**: MySQL for storing and managing application data.
3. **Proxy**: Nginx acting as a reverse proxy for the backend, secured with HTTPS.

![Untitled-5-01](https://github.com/user-attachments/assets/1fc37ab7-fb9d-4f31-819b-0eece5aecad7)

## Technologies Used

- **Go**: The backend API is developed in Go.
- **MySQL**: Used as the database for persisting data.
- **Nginx**: Used as a reverse proxy to route traffic to the backend, serving over HTTPS.
- **Docker**: Containers are used to manage the services.
- **Docker Compose**: Orchestrates the multi-container environment.
- **SSL**: The project includes SSL configuration for Nginx to serve over HTTPS

## Prerequisites

- [Docker](https://www.docker.com/)
- [Docker Compose](https://docs.docker.com/compose/install/)
- OpenSSL (for generating SSL certificates)

## Getting Started

# Three-Tier Application with Docker

This project involves the development of a three-tier architecture application that consists of a backend service (written in Go), a MySQL database for data persistence, and an Nginx proxy acting as a reverse proxy to route traffic to the backend over HTTPS. The project was built using Docker principles to ensure containerization, security, and easy orchestration.

## Project Development Overview

### 1. Setting up the Backend (Go Application)

The first step was to develop the backend service using Go. The backend handles business logic and interacts with the MySQL database. We created a basic HTTP server in Go that listens for requests and performs basic operations, such as database queries. We used a Go module to manage dependencies (`go.mod` and `go.sum`).

A multi-stage Dockerfile was written for the Go application to optimize the final image size. The first stage used the official Go image to build the binary, and the second stage copied the binary to a minimal base image (`scratch` or `alpine`). The Dockerfile looks like this:

```Dockerfile
FROM golang:alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o main .

FROM alpine:latest
WORKDIR /root/
COPY --from=builder /app/main .
CMD ["./main"]
```

This Dockerfile ensures that the final image only contains the Go binary and required libraries, minimizing the size.

### 2. Database Layer (MySQL)
Next, we set up the MySQL database. The database is used to store and manage application data. In a production environment, securing database credentials is crucial. Therefore, the MySQL root password and other sensitive information were stored in a file on the host machine (mysql/db-pass), which was passed to the MySQL container as a Docker secret.

A MySQL Docker image was used to spin up the database, and the secret (db-pass) was mounted inside the container to supply the root password. The Docker Compose configuration ensures that the backend can access the database through the internal Docker network.

```Docker-compose mysql-db service
mysql-db:
  image: mysql:latest
  environment:
    MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db-pass
  secrets:
    - db-pass
  networks:
    - backend-network
  volumes:
    - ./mysql/db-password:/run/secrets/db-pass
```


The backend connects to the database using the root password stored in the secret.

### 3. SSL Certificates and Security
For securing the application, SSL certificates were generated using OpenSSL and placed in the nginx/ssl/ directory. The certificates were mounted into the Nginx container, enabling HTTPS communication. While self-signed certificates are used for development, in production, we would use certificates from a trusted Certificate Authority (CA).\

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout nginx/ssl/nginx.key -out nginx/ssl/nginx.crt
```
This command creates an SSL certificate and private key, which are then used by Nginx to serve HTTPS traffic.

### 4.Setting up the Proxy Layer (Nginx)
To ensure secure communication, we set up Nginx as a reverse proxy that handles HTTPS traffic. First, we generated SSL certificates using OpenSSL and stored them in the nginx/ssl/ directory on the host machine. These certificates were then mounted into the Nginx container.

The Nginx configuration (nginx/conf.d/default.conf) was set to listen on port 443 for HTTPS traffic. Nginx forwards all incoming requests to the Go backend running on port 8000. The proxy configuration also includes SSL settings, specifying the certificate and private key files. Below is an excerpt of the Nginx configuration:

```Nginx conf custom file
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    location / {
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```Nginx Dockerfile
FROM nginx:alpine
COPY default.conf /etc/nginx/conf.d/default.conf
RUN  mkdir -p /etc/nginx/ssl
COPY /ssl/nginx.crt /etc/nginx/ssl
COPY /ssl/nginx.key /etc/nginx/ssl
```
This configuration ensures that Nginx listens for secure HTTPS connections, forwards them to the backend service, and passes important client information through headers.

### 5. Creating Docker Networks for Isolation
Each service was placed in its own Docker network for security and isolation. The backend and MySQL database were connected through a private network called backend-network, while Nginx had its own network to interact with the outside world. This network separation ensures that services only communicate with the necessary components, reducing the attack surface.

The network configuration in docker-compose.yml looked like this:

```
networks:
  backend-network:
    driver: bridge
  proxy-network:
    driver: bridge
  db-network:
    driver: bridge
```
### 6. Docker Compose for Orchestration
To manage and orchestrate the containers, a docker-compose.yml file was created. This file defines the backend, database, and Nginx services, as well as the necessary environment variables, secrets, and network connections.

The Compose file allows us to bring the entire application up or down with a single command (docker-compose up or docker-compose down). This simplifies the development process by automating container management.   


### 7. Testing and Troubleshooting
After setting up the environment, we tested the application by accessing it via https://localhost and http://localhost:8000. The Nginx proxy was up and running, forwarding requests to the Go backend, which interacted with the MySQL database. Troubleshooting involved checking logs for each container (docker-compose logs <service>) and ensuring that the correct networks and secrets were configured.