---
title: "Testing the performance of a ML endpoint using Locust, FastAPI and Docker"
date: 2022-07-28T10:54:58+02:00
draft: true
---

## Introduction

Recently, I implemented a local deployment solution in one of my side projects, which consists of an application to detect and classify the waste from an image. The local deployment has been made using Gradio, FastAPI and Docker:

![local_deployment](/local_deployment.png)

All the code related to the deployment can be found [here](https://github.com/hectorLop/Waste-Detector/tree/main/local_deployment). Besides, both containers are managed using docker-compose, using the following configuration:

```yaml
version: '3'
services:
  backend:
      build:
          context: .
          dockerfile: backend.Dockerfile 
      image: waste_detector_backend
      container_name: waste_detector_backend
      ports:
          - "5000:5000"
      command: uvicorn app:app --host 0.0.0.0 --port 5000
  frontend:
      restart: always
      build:
          context: .
          dockerfile: frontend.Dockerfile
      image: waste_detector_frontend
      container_name: waste_detector_frontend
      ports:
          - "8501:8501"
      command: python3 -m deployment.frontend
```

<br>

Now, the idea is to test the performance of this application. Thus we can get an idea of how would the application behave in a production environment.

<br>

### Load Testing using Locust

[Locust](https://locust.io/) is a load testing tool and framework written in and controlled using Python. It allows writing the user behaviors and specifying the number of user you would like to simulate. It can scale to massive applications too.

Beyond these features, Locust also allows to export the statistics of your test for further analysis in CSV format. For another, it has a simple user interface (UI) too. The UI allows you to configure the number of users, the host you would like to test and set the rate at which you would like to add users to the simulation.

Locust gets information of the number of requests per second (RPS) the service is currently handling (a throughput statistic). It also gets statistics on the average, median, minimum and maximum latency in milliseconds.
