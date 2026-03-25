# Expose Wordsmith using Traefik

This guide shows you how to expose the Docker sample application [Wordsmith](https://github.com/dockersamples/wordsmith) using Traefik to an HTTPS connection. The demo project consists of a web and an API service. In this guide, we will add the following configurations:
- The web service is protected with basic authentication (username and password).
- The API service is protected with rate limiting.

## Prerequisites

- Install Docker and Docker Compose.
- Deploy Traefik. Refer to the [Docker setup guide](https://doc.traefik.io/traefik/setup/docker/) for instructions.

## 1. Clone the Wordsmith repository

In a directory of your choice, clone the Wordsmith repository using the following command:

```bash
git clone https://github.com/dockersamples/wordsmith.git
```

The `wordsmith` folder is added to your working directory.

## 2.Get self-signed certificates

This is ideal for local development and testing purposes. Create a folder called `certs` in your working directory which stores the SSL certificates. Use the following command to generate self-signed certificates:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048   -keyout certs/local.key -out certs/local.crt   -subj "/CN=*.localhost"
```

Note the `*.localhost` in the command. This is the domain name that we will use to access the web and API services. You can change this according to your needs. 

Next, you must also create a `traefik-dynamic.yml` file to add the certificates that will be referenced in the `docker-compose.yml` file. In the `traefik-dynamic.yml` file, enter the following:

```yaml
tls:
  certificates:
    - certFile: /certs/local.crt
      keyFile: /certs/local.key
```

## 3. Configure Traefik

In the `wordsmith` folder, a `docker-compose.yml` file is present. It contains the configuration for the database, web, and API services. Edit the file as follows:

```yaml
version: '3.9'
# we'll keep the version for now to work in Compose and Swarm
# compose file with self-signed certificates

services:
  traefik:
    image: traefik:v3
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./traefik-dynamic.yml:/etc/traefik/dynamic.yml:ro"
      - "./certs/local.crt:/certs/local.crt:ro"
      - "./certs/local.key:/certs/local.key:ro"
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--api.dashboard=true"
      - "--log.level=DEBUG"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      - "--providers.file.filename=/etc/traefik/dynamic.yml"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.localhost`)" # This is the domain name to access the Traefik dashboard. Change it if you want to use a different domain name.
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls=true"
      - "traefik.http.routers.dashboard.service=api@internal"

  db:
    image: postgres:10.0-alpine
    volumes:
      - ./db:/docker-entrypoint-initdb.d/

  api:
    build: api
    image: dockersamples/wordsmith-api
    deploy:
      replicas: 5
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`wordsmith-api.localhost`)" # This is the domain name to access the API service. Change it if you want to use a different domain name.
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.routers.api.tls=true"
      - "traefik.http.services.api.loadbalancer.server.port=8080"
      - "traefik.http.middlewares.api-ratelimit.ratelimit.average=100" # Applying rate limiting
      - "traefik.http.middlewares.api-ratelimit.ratelimit.burst=50"
      - "traefik.http.routers.api.middlewares=api-ratelimit"

  web:
    build: web
    image: dockersamples/wordsmith-web
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.web.rule=Host(`wordsmith-web.localhost`)" # This is the domain name to access the web service. Change it if you want to use a different domain name.
      - "traefik.http.routers.web.entrypoints=websecure"
      - "traefik.http.routers.web.tls=true"
      - "traefik.http.services.web.loadbalancer.server.port=80"
      - "traefik.http.middlewares.web-auth.basicauth.users=admin:$$apr1$$zl8sgpeM$$LZY81PJYrmHZp701vs9Ct0" # Applying basic auth. Get the hash password using the command `htpassword -nb admin <password>`. Replace `admin` with the username and `<password>` with the password. Remember to escape the dollar symbols with `$$`.
      - "traefik.http.routers.web.middlewares=web-auth"
```

This file defines the following:
- `traefik` service: This is used to expose the web and API services to the internet. It is configured to use HTTPS and requires a valid certificate to access the services. Additionally, the Traefik dashboard is also exposed via HTTPS.
- `db`, `api`, and `web` services: These are the services that make up the Wordsmith application. The API service is rate-limited to 100 requests per second and the web service is protected with basic authentication by incorporating [middlewares](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/overview/).

## 4. Deploy the application

Deploy the application using the following command:

```bash
docker compose up -d
```

You should be able to access the web, API, and Traefik dashboard services using the URLs that you defined in the `docker-compose.yml` file. Because we are using self-signed certificates, you will need to accept the security warnings in your browser to access the services.

When you are accessing the web service, you will be prompted to enter a username and password (`admin`/`admin`). To access the API service, enter the domain name followed by `/nouns`, `/verbs`, and `/adjectives`. A JSON object is returned that contains a randomly generated word based on the application's database.

The following screenshot shows how an example of the Traefik dashboard:

![Traefik dashboard](/assets/images/docker-traefik-dashboard")

---

## Kubernetes setup

If you are using Kubernetes, the initial steps are the same as for Docker. Have a cluster ready, clone the Wordsmith repository, and get the self-signed certificates. 

The Wordsmith project provides the [Kustomize configuration](https://github.com/dockersamples/wordsmith/blob/main/kustomization.yaml). It defines the necessary deployment and service objects.

1. In the `wordsmith` folder, generate the TLS secret from the certificates provided:

```bash
kubectl create secret tls local-cert --cert=certs/local.crt --key=certs/local.key
```

2. In the cloned repository, add the Traefik Helm chart by using the following commands:

```bash
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
helm install traefik traefik/traefik -n default
```

2. Create a `traefik-infrastructure.yml` file to define the TLS secret, middlewares, and ingress routes:

```yaml
# 0. Set auth secret for the web service
apiVersion: v1
kind: Secret
metadata:
  name: authsecret
  namespace: default
type: Opaque
stringData:
  users: |
    admin:$apr1$vonb2qec$Q1hTEPsgJxjoJzaLscx251
---
# 1. Set the default fallback TLS certificate
apiVersion: traefik.io/v1alpha1
kind: TLSStore
metadata:
  name: default
  namespace: default
spec:
  defaultCertificate:
    secretName: local-cert
---
# 2. Define the middlewares (auth & rate limit)
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: web-auth
spec:
  basicAuth:
    secret: authsecret
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: api-ratelimit
spec:
  rateLimit:
    average: 100
    burst: 50
---
# 3. Route the web service
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: wordsmith-web-ingress
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`wordsmith-web.localhost`)
      kind: Rule
      services:
        - name: web        
          port: 8080       
      middlewares:
        - name: web-auth   # adding the middleware
---
# 4. Route the API service
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: wordsmith-api-ingress
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`wordsmith-api.localhost`)
      kind: Rule
      services:
        - name: api      
          port: 8080      
      middlewares:
        - name: api-ratelimit # adding the middleware
---
# 5. Route the Traefik Dashboard
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`traefik.localhost`)
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
```

6. Deploy the application using the following commands:

```bash
kubectl apply -k .
kubectl apply -f traefik-infrastructure.yml
```

You can check the status of the pods using the command: `kubectl get pods`. You should now be able to access the web, API, and Traefik dashboard services using the URLs that you defined in the `traefik-infrastructure.yml` file.

## Troubleshooting

### Errors with Kubernetes pods

1. In the Kubernetes deployment method, if you are using Windows and Docker Desktop, you may get errors with Traefik and the Wordsmith database deployment. To fix this, manually pull the failing images using the following commands:

```bash
docker pull postgres:10.0-alpine
docker pull traefik:v3.6.11
```

Then delete the failed pods instances:

```bash
kubectl delete pod -l app=db
kubectl delete pod -l app.kubernetes.io/name=traefik -n default
```

Check the status of the pods again to verify that they are running. If the issue persists, you may need to restart Docker Desktop.

### Cannot access the services and 404 page not found error

There can be multiple reasons for the errors. Make sure that the self-signed certificates are generated and available in the path that you have defined in the configuration files. You can check the Traefik logs.

- For Docker, check the logs using the command: `docker logs traefik`
- For Kubernetes, check the logs using the command: `kubectl logs -n default -l app.kubernetes.io/name=traefik`
