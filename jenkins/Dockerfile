# Jenkins LTS with Python added so the pipeline can create a venv and run pytest
FROM jenkins/jenkins:lts-jdk17

USER root
RUN apt-get update && \
    apt-get install -y --no-install-recommends python3 python3-pip python3-venv curl && \
    rm -rf /var/lib/apt/lists/*
USER jenkins

# Plugins needed for this assignment
RUN jenkins-plugin-cli --plugins \
    git \
    github \
    workflow-aggregator \
    junit \
    credentials-binding \
    timestamper
