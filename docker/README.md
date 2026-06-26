# Docker - Containerization Platform

## What is Docker?
Docker is a containerization platform that packages applications with all dependencies into a standardized unit called a container. Containers ensure consistency across development, testing, and production environments.

---

## Real-Time Scenario 1: Development to Production Consistency

### Scenario Description
A developer builds a Node.js application on macOS with Node 14.x and MongoDB 4.x. When deployed to a Linux production server, the app crashes because the server has Node 12.x and MongoDB 3.x installed.

### Problem (Without Docker)
```
Developer's Machine:          Production Server:
Node 14.x                     Node 12.x
MongoDB 4.x                   MongoDB 3.x
Redis 6.x                     Redis 5.x
Python 3.9                    Python 3.6

Result: "Works on my machine!" fails in production
```

### Docker Solution
```dockerfile
# Dockerfile - Ensures exact environment everywhere
FROM node:14-alpine

WORKDIR /app

# Copy application files
COPY package*.json ./
RUN npm install

COPY . .

# Copy MongoDB and Redis config
COPY config/ /etc/config/

EXPOSE 3000

CMD ["node", "server.js"]
```

### Build and Run
```bash
# Build image
docker build -t myapp:1.0 .

# Run container
docker run -p 3000:3000 \
  -e DB_HOST=mongodb \
  -e NODE_ENV=production \
  myapp:1.0
```

### Result
- Developer's laptop → Same container ✅
- QA testing → Same container ✅
- Production server → Same container ✅
- CI/CD pipeline → Same container ✅

---

## Real-Time Scenario 2: Microservices Architecture

### Scenario Description
An online learning platform has multiple services:
- User authentication service (Python)
- Course catalog service (Node.js)
- Payment service (Java)
- Notification service (Go)
- Video streaming service (C++)

All run on the same server without conflicts.

### Problem Without Docker
- Python 3.8 conflicts with Python 2.7
- Java 8 vs Java 11 compatibility issues
- Different port requirements cause conflicts
- Version management nightmare

### Docker Solution - docker-compose.yml
```yaml
version: '3.8'

services:
  auth-service:
    build: ./auth
    image: learning-auth:1.0
    ports:
      - "5001:5000"
    environment:
      - FLASK_ENV=production
      - DATABASE_URL=postgresql://db:5432/auth
    depends_on:
      - db

  catalog-service:
    build: ./catalog
    image: learning-catalog:1.0
    ports:
      - "3001:3000"
    environment:
      - NODE_ENV=production
      - DB_URL=mongodb://mongodb:27017
    depends_on:
      - mongodb

  payment-service:
    build: ./payment
    image: learning-payment:1.0
    ports:
      - "8001:8080"
    environment:
      - JAVA_OPTS=-Xmx512m
    depends_on:
      - db

  notification-service:
    build: ./notification
    image: learning-notification:1.0
    ports:
      - "7001:7000"
    environment:
      - GO_ENV=production

  video-stream:
    build: ./video
    image: learning-video:1.0
    ports:
      - "6001:6000"
    volumes:
      - /data/videos:/videos
    environment:
      - VIDEO_PATH=/videos

  db:
    image: postgres:13
    environment:
      - POSTGRES_DB=learning
      - POSTGRES_PASSWORD=secret
    volumes:
      - pgdata:/var/lib/postgresql/data

  mongodb:
    image: mongo:4
    volumes:
      - mongodata:/data/db

volumes:
  pgdata:
  mongodata:
```

### Run Everything
```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f

# Stop all services
docker-compose down
```

### Benefits
- Each service isolated in its own container
- No port conflicts (mapped to different ports)
- Easy to add/remove services
- Automatic networking between containers
- One command to start entire platform

---

## Real-Time Scenario 3: CI/CD Pipeline Consistency

### Scenario Description
A company runs tests in Jenkins CI. Tests pass on Jenkins but fail on developer machines because Jenkins uses different dependencies.

### Problem
- CI passes → Production fails
- Debugging CI issues is hard
- Can't reproduce locally

### Docker Solution - Jenkins Pipeline
```groovy
pipeline {
    agent {
        docker {
            image 'python:3.9-slim'
            args '-v /data:/data'
        }
    }
    
    stages {
        stage('Install Dependencies') {
            steps {
                sh '''
                    pip install -r requirements.txt
                    pip install pytest pytest-cov
                '''
            }
        }
        
        stage('Run Unit Tests') {
            steps {
                sh 'pytest tests/ -v'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t myapp:${BUILD_NUMBER} .
                    docker tag myapp:${BUILD_NUMBER} myapp:latest
                '''
            }
        }
        
        stage('Push to Registry') {
            steps {
                sh '''
                    docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}
                    docker push myapp:${BUILD_NUMBER}
                '''
            }
        }
    }
}
```

### Pipeline Flow
1. Jenkins pulls Python 3.9 container
2. Installs exact dependencies
3. Runs tests in container
4. Builds application image
5. Pushes to Docker Registry
6. Kubernetes deploys from registry

### Result
- Same environment everywhere
- CI success = Production success
- No "works on Jenkins, not on my machine" issues

---

## Real-Time Scenario 4: Zero-Downtime Deployments

### Scenario Description
Deploy new version while old version handles traffic.

### Docker Solution with Load Balancer
```bash
# Terminal 1: Old version running
$ docker run -d -p 8001:3000 --name app-v1 myapp:1.0

# Terminal 2: New version running
$ docker run -d -p 8002:3000 --name app-v2 myapp:2.0

# Nginx routes traffic to both:
# 50% → app-v1 (port 8001)
# 50% → app-v2 (port 8002)

# After verification, stop old version:
$ docker stop app-v1

# Route 100% traffic to app-v2
```

---

## Docker Concepts

### Image
- Blueprint for containers
- Contains OS, dependencies, code
- Immutable

### Container
- Running instance of image
- Has isolated filesystem, network, process space
- Lightweight (MB not GB)

### Registry
- Repository for images (Docker Hub, ECR, GCR)
- Push/pull images from anywhere

### Dockerfile
- Instructions to build image
- Declarative (what, not how)

---

## When to Use Docker

✅ **Good Use Cases:**
- Microservices
- Need consistency across environments
- Rapid deployment
- Complex dependencies
- Multi-language stack

❌ **Avoid If:**
- Simple single-server application
- GUI applications
- Real-time systems with strict latency

---

## Learning Path
1. Learn basic Docker commands
2. Create Dockerfile for simple app
3. Use docker-compose for multiple services
4. Push/pull images from registry
5. Understand networking between containers
6. Practice volume management
