FROM jenkins/jenkins:lts-jdk17
USER root
# Client Docker et git : le démon reste celui de Docker Desktop (via le socket monté)
RUN apt-get update && apt-get install -y docker.io git && rm -rf /var/lib/apt/lists/*