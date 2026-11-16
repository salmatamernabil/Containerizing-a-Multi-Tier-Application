# VProfile – Multi-Tier Application Containerization with Docker & Harbor

## Project Overview

This project demonstrates the containerization and deployment of the VProfile multi-tier Java web application using Docker and Docker Compose.

The application consists of multiple interconnected services, each running in its own container. Docker Compose is used to orchestrate the services, configure networking, and manage persistent storage.

Additionally, Harbor Container Registry is used as a private repository to store and manage the container images, allowing authenticated users to push and pull application images.

## Architecture

The infrastructure consists of the following components:

| Service | Container Name | Technology | Purpose |
|---|---|---|---|
| Database | vprodb | MySQL 8.0.33 | Stores application data |
| Cache | vprocache01 | Memcached | Application caching |
| Message Broker | vpromq01 | RabbitMQ | Message queuing |
| Application | vproapp | Apache Tomcat 9 | Hosts the Java application |
| Web Server | vproweb | NGINX | Reverse proxy for incoming requests |
| Image Registry | Harbor | Harbor Registry | Private Docker image storage |

## Project Structure

```text
VProfile/
│
├── Docker-compose.yml
│
└── Docker-files/
    │
    ├── app/
    │   └── Dockerfile
    │
    ├── db/
    │   ├── Dockerfile
    │   └── db_backup.sql
    │
    └── web/
        ├── Dockerfile
        └── nginvproapp.conf
```

## 1. Application Containerization

The Java application is built using a multi-stage Dockerfile.

### Application Dockerfile

The first stage uses Maven with Eclipse Temurin JDK 17 to clone the VProfile repository, switch to the Docker branch, and build the application.

The second stage uses Tomcat 9 with JRE 11 to deploy the generated WAR file.

```dockerfile
FROM maven:3.9.6-eclipse-temurin-17 AS build_image

RUN apt update && apt install git -y

RUN git clone https://github.com/devopshydclub/vprofile-project.git

RUN cd vprofile-project && \
    git checkout docker && \
    mvn install

FROM tomcat:9-jre11

LABEL "Project"="Vprofile"

RUN rm -rf /usr/local/tomcat/webapps/*

COPY --from=build_image \
    vprofile-project/target/vprofile-v2.war \
    /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080

CMD ["catalina.sh", "run"]
```

**Build process:**

1. Clone the application source code.
2. Checkout the Docker branch.
3. Build the Java application using Maven.
4. Generate the WAR artifact.
5. Copy the WAR file into the Tomcat container.
6. Deploy the application as the ROOT web application.

This multi-stage build separates the build environment from the runtime environment.

## 2. Database Containerization

MySQL 8.0.33 is used as the application database.

### Database Dockerfile

```dockerfile
FROM mysql:8.0.33

LABEL "Project"="Vprofile"

ENV MYSQL_ROOT_PASSWORD="vprodbpass"
ENV MYSQL_DATABASE="accounts"

ADD db_backup.sql /docker-entrypoint-initdb.d/db_backup.sql
```

The database configuration:

- Uses MySQL 8.0.33.
- Sets the root password.
- Creates the `accounts` database.
- Imports the initial database backup through the MySQL initialization directory.

The database data is stored in a Docker named volume to maintain persistence across container restarts and recreation.

## 3. NGINX Reverse Proxy Configuration

NGINX is deployed as the frontend web server and reverse proxy.

### NGINX Dockerfile

```dockerfile
FROM nginx

LABEL "Project"="Vprofile"

RUN rm -rf /etc/nginx/conf.d/default.conf

COPY nginvproapp.conf /etc/nginx/conf.d/vproapp.conf
```

### NGINX Configuration

```nginx
upstream vproapp {
    server vproapp:8080;
}

server {
    listen 80;

    location / {
        proxy_pass http://vproapp;
    }
}
```

NGINX listens on port 80 and forwards incoming HTTP requests to the application container on port 8080.

Docker Compose service discovery allows NGINX to resolve the application using the service name `vproapp`.

## 4. Docker Compose Configuration

Docker Compose is used to define and orchestrate all application services.

The infrastructure includes:

- MySQL database.
- Memcached cache.
- RabbitMQ message broker.
- Tomcat application server.
- NGINX reverse proxy.

### Starting the Application

```bash
docker compose up -d
```

### Checking Running Containers

```bash
docker compose ps
```

### Viewing Logs

```bash
docker compose logs -f
```

### Checking a Specific Service

```bash
docker compose logs -f vproapp
```

### Stopping the Application

```bash
docker compose down
```

### Persistent Volumes

Two named volumes are configured:

| Volume | Mount Point | Purpose |
|---|---|---|
| vprodbdata | `/var/lib/mysql` | Persistent MySQL database storage |
| vproappdata | `/usr/local/tomcat/webapps` | Persistent Tomcat web application directory |

These volumes help retain data independently of the container lifecycle.

## 5. Deploying and Testing the Application

After starting the containers, verify that all services are running:

```bash
docker compose ps
```

Access the application through the NGINX reverse proxy:

```text
http://<DOCKER-HOST-IP>
```

The application is exposed through NGINX on port 80, which forwards requests to Tomcat.

The application container also exposes port 8080 for direct testing.

## 6. Harbor Private Container Registry

Harbor was used to store the Docker images in a private project.

The objective was to build the application images, upload them to Harbor, configure private repository access, and retrieve the images from the registry.

### Step 1: Create a Private Harbor Project

1. Access the Harbor web interface.
2. Log in using Harbor credentials.
3. Navigate to **Projects**.
4. Create a new project.
5. Configure the project visibility as **Private**.

### Step 2: Login to Harbor

Authenticate Docker against the Harbor registry:

```bash
docker login <HARBOR-IP-OR-HOSTNAME>
```

Enter the Harbor username and password.

### Step 3: Build the Docker Images

Build the application images:

```bash
docker compose build
```

Verify the generated images:

```bash
docker images
```

### Step 4: Tag Images for Harbor

Tag the images using the Harbor registry address and private project name.

```bash
docker tag vprocontainers/vprofiledb \
<HARBOR-IP-OR-HOSTNAME>/<PROJECT-NAME>/vprofiledb:latest

docker tag vprocontainers/vprofileapp \
<HARBOR-IP-OR-HOSTNAME>/<PROJECT-NAME>/vprofileapp:latest

docker tag vprocontainers/vprofileweb \
<HARBOR-IP-OR-HOSTNAME>/<PROJECT-NAME>/vprofileweb:latest
```

### Step 5: Push Images to Harbor

```bash
docker push <HARBOR-IP-OR-HOSTNAME>/<PROJECT-NAME>/vprofiledb:latest

docker push <HARBOR-IP-OR-HOSTNAME>/<PROJECT-NAME>/vprofileapp:latest

docker push <HARBOR-IP-OR-HOSTNAME>/<PROJECT-NAME>/vprofileweb:latest
```

### Step 6: Verify Uploaded Images

In the Harbor web interface:

1. Open the private project.
2. Navigate to Repositories.
3. Verify that the database, application, and web server images are available.

### Step 7: Pull Images from Private Harbor

To verify image retrieval, download the images from Harbor:

```bash
docker pull <HARBOR-IP-OR-HOSTNAME>/<PROJECT-NAME>/vprofiledb:latest

docker pull <HARBOR-IP-OR-HOSTNAME>/<PROJECT-NAME>/vprofileapp:latest

docker pull <HARBOR-IP-OR-HOSTNAME>/<PROJECT-NAME>/vprofileweb:latest
```

Verify the downloaded images:

```bash
docker images
```

This demonstrates that the images are stored centrally and can be retrieved from the private registry by authenticated Docker clients.

