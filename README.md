# traefik-challenge

This repository contains the files for the Traefik challenge scenario.

### List of files present in the repository

- [documentation.md](documentation.md): This file contains the complete guide for exposing a sample project on HTTPs, with basic auth and rate applied. I've chosen a Docker demo project (Wordsmith) to demonstrate the scenario. This assumes that the user wants to expose the Wordsmith project with self-signed certificates. The steps for Kubernetes are also included. Configuration for production use with Let's Encrypt is not included because I had no means of testing the steps. I had some issues with the Traefik version to be used (`v3.0` vs `v3`) and with the Kubernetes deployment, so I added a "Troubleshooting" section with a couple of error scenarios and solutions. 

- [assets](assets): This folder contains screenshots of the Traefik dashboard from the Docker and Kubernetes deployments.

- [docker-compose.yml](docker-compose.yml) and [traefik-dynamic.yml](traefik-dynamic.yml): These files contain the configuration for Docker deployment that I used.

- [traefik-infrastructure.yml](traefik-infrastructure.yml): This file contains the configuration for Kubernetes.

### Tools used

- I'm using a Windows machine with Docker Desktop and Git Bash.
- Google Antigravity was used to write the configuration files and for troubleshooting deployment and run-time errors.

### Method

1. Discovery phase: I went over the Traefik docs for Docker initially, since that was my primary target for completion. I completed the basic setup and went over the code snippets to better understand the structure for exposure, basic auth, and middlewares in general. Here, I also looked for a project for implementation and chose Wordsmith for its simplicity and web and API services that aligned with the goal of exposing two services.

2. Testing phase: In this phase, I aimed to add basic auth and rate limit to the Wordsmith project. With some trial and error (some of it because of Windows), the deployment was successful. Then I went over the Traefik Kubernetes implementation as well and repeated the configuration and testing to successfully deploy the services.

3. Writing phase: With my notes from the previous phases, I worked on an initial structure of the guide and added in the details. Since the aim was to only add the Kubernetes changes, I kept the section at the end. I tried to keep it to the point since the reader is a platform engineer and would have the basic knowledge.

4. Editing and re-testing phase: Reviewed the guide for clarity, consistency, and general flow. I also tested the steps again, following the steps from the guide itself to make sure it's repeatable.

I hope the reader is successful in deploying the services!
