# Real-Time WebSocket Chat App - DevOps Deployment

A containerized real-time chat application deployed with Docker, Nginx, and automated via GitHub Actions CI/CD.

Live URL: http://3.25.23.224

## Project Overview

This project takes a pre-built FastAPI WebSocket chat backend and a static HTML frontend, and deploys them using a production-style setup: Docker containers, an Nginx reverse proxy, and a fully automated CI/CD pipeline on AWS EC2.

## Architecture

User Browser -> Public IP (AWS EC2, port 80) -> NGINX Container (reverse proxy) -> serves static frontend AND proxies /ws to -> FastAPI Backend Container (port 8000, WebSocket)

Two containers run on a shared Docker bridge network (chat-network):

- nginx - listens on port 80, serves the static frontend from /frontend, and reverse-proxies all /ws traffic to the backend.
- backend - a FastAPI app running with uvicorn, exposing port 8000 internally, handling WebSocket connections at /ws.

## How Docker Networking Works

Both containers join a custom bridge network called chat-network, defined in docker-compose.yml. Docker Compose automatically gives each container a DNS entry matching its service name. This means the nginx container can reach the backend simply by using the hostname backend, e.g. http://backend:8000 - it does not need to know the backend container's actual IP address.

## How Nginx Reverse Proxy Works

Nginx listens on port 80, the only port exposed to the internet, and does two jobs:

1. Serves static files - the location / block serves index.html and other frontend assets from /usr/share/nginx/html, which is volume-mounted from the local ./frontend folder.
2. Reverse-proxies WebSocket traffic - the location /ws block forwards any request to /ws to the backend container at http://backend:8000/ws.

## How WebSocket Works Through Nginx

WebSocket connections start as a normal HTTP request that asks to be upgraded to a persistent WebSocket connection. For Nginx to allow this upgrade instead of treating it as a normal HTTP request, it must forward two specific headers to the backend: Upgrade and Connection: upgrade. Without these headers, the handshake fails silently and the frontend stays stuck on Disconnected.

## How CI/CD Pipeline Works

A GitHub Actions workflow (.github/workflows/deploy.yml) is triggered on every push to the main branch. It:

1. Connects to the EC2 server over SSH using credentials stored as encrypted GitHub Secrets (EC2_HOST, EC2_SSH_KEY).
2. Runs git pull origin main on the server to fetch the latest code.
3. Runs docker compose up -d --build to rebuild and restart the containers with the new code.

This means any code change pushed to GitHub is automatically live on the server within seconds, with no manual server login required.

## Issues Found and How They Were Fixed

1. Backend bound to 127.0.0.1 instead of 0.0.0.0 - In Dockerfile, uvicorn was started with --host 127.0.0.1, which only accepts connections from inside its own container. Fix: changed to --host 0.0.0.0.

2. Frontend volume mount was commented out - In docker-compose.yml, the line mounting ./frontend into the nginx container's web root was commented out. Fix: uncommented it.

3. Nginx WebSocket proxy misconfigured - proxy_pass pointed to localhost:8000 instead of backend:8000, and the Upgrade/Connection headers were commented out. Fix: corrected proxy_pass and enabled the headers.

An explicit chat-network bridge network was also added to docker-compose.yml for clearer container networking.

## Deployment Steps

git clone https://github.com/sireesha-droid/devops.git
cd devops
docker compose up -d --build

Then visit http://<server-ip> in a browser.

## Tech Stack

Backend: FastAPI + Uvicorn (Python)
Frontend: Static HTML/JS
Reverse Proxy: Nginx (Alpine)
Containerization: Docker + Docker Compose
CI/CD: GitHub Actions
Cloud: AWS EC2 (Free Tier)
