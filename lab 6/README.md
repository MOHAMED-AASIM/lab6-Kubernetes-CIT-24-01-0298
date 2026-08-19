# CCS3308 Lab 6 — Kubernetes Fundamentals with Minikube

## Overview
This lab deploys a multi-tier application (nginx frontend, httpbin API, redis cache, postgres database) on a local Minikube Kubernetes cluster. All images are public Docker Hub images — no custom application code.

## Architecture
Frontend (Deployment, NodePort) → API (Deployment, ClusterIP) → Cache (Deployment, ClusterIP) → Database (StatefulSet + PVC, Headless Service)

## How to Run
1. minikube start --driver=docker
2. kubectl apply -f k8s/
3. minikube service frontend --url

## Files
- k8s/ — all Kubernetes manifests
- screenshots/ — evidence for each task (task1.1 - task10.1)
- answers.md — checkpoint question answers

## Notes
- VM had limited RAM (~3.3GB), so minikube was started with --memory=2200mb --cpus=2 for stability.
- Slow internet connection caused image pulls (especially postgres:16-alpine) to take significantly longer than expected — postgres-0 stayed in ContainerCreating for ~20+ minutes during Part 7/8 due to concurrent image pulls competing for bandwidth.
- Encountered a gcr.io DNS resolution issue when pulling the minikube kicbase image; resolved by configuring DNS to use Google DNS (8.8.8.8/1.1.1.1), after which minikube fell back to the docker.io mirror successfully.
