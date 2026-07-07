To make it super easy to copy everything at once, I have wrapped the entire guide inside a single code block below.

Just hover your mouse over the top right corner of the block below, and a small **"Copy" icon** (it usually looks like two overlapping squares) will appear. Click that, and the entire text will be copied to your clipboard, ready to be pasted directly into your `DOCKER_SETUP.md` file!

````markdown
# 🐳 Docker Setup Guide for Node.js Backend

This guide provides step-by-step instructions for dockerizing this Node.js/TypeScript backend application. It covers creating the Docker image, setting up Docker Compose for local development, and pushing the image to a container registry.

## Prerequisites

- Docker installed on your machine.
- Docker Compose installed (usually bundled with Docker Desktop).
- A `.env` file in the root of your project containing your local environment variables.

---

## Step 1: Ignore Unnecessary Files

Create a `.dockerignore` file in the root of your project. This prevents large or sensitive files from being copied into the Docker image, keeping the image lightweight, fast to build, and secure.

```text
node_modules
npm-debug.log
dist
.env
.git
.gitignore
.ai
README.md
```
````

---

## Step 2: Create the Dockerfile

Create a `Dockerfile` in the root of your project. This configuration uses a multi-stage build process. First, it compiles the TypeScript code, and then it packs only the necessary production files into the final image to keep the size small.

```dockerfile
# ==========================================
# Stage 1: Build the application
# ==========================================
FROM node:18-alpine AS builder

# Set the working directory
WORKDIR /usr/src/app

# Copy dependency definition files
COPY package*.json ./

# Install all dependencies (including dev dependencies required for building)
RUN npm ci

# Copy the rest of the application code
COPY . .

# Build the code (assumes a "build" script exists in package.json)
RUN npm run build

# ==========================================
# Stage 2: Run the production application
# ==========================================
FROM node:18-alpine

WORKDIR /usr/src/app

# Copy only dependency files
COPY package*.json ./

# Install only production dependencies
RUN npm ci --omit=dev

# Copy the compiled output from the builder stage
COPY --from=builder /usr/src/app/dist ./dist

# Expose the port the app runs on (update if your app uses a different port)
EXPOSE 5000

# Start the application
CMD ["node", "dist/server.js"]

```

---

## Step 3: Setup Docker Compose (Local Development)

Create a `docker-compose.yml` file in the root of your project. This allows you to easily run, network, and manage the container locally with a single, reproducible configuration.

```yaml
services:
  api:
    # Replace <your-dockerhub-username> and <your-app-name> with your actual details
    image: <your-dockerhub-username>/<your-app-name>:latest
    container_name: backend-api
    ports:
      # Map HostPort:ContainerPort
      - "5000:5000"
    env_file:
      - .env
    restart: unless-stopped
```

---

## Step 4: Running the Application Locally

Use the following commands in your terminal to manage your application locally using Docker Compose. Make sure you are in the directory containing the `docker-compose.yml` file.

**Start the application in the background (detached mode):**

```bash
docker-compose up -d

```

**View live logs for the container:**

```bash
docker-compose logs -f

```

**Stop and completely remove the container:**

```bash
docker-compose down

```

---

## Step 5: Building and Pushing to a Registry

If you need to deploy this image to a remote production server or share it with others, you can build and push it to a container registry (like Docker Hub, AWS ECR, or Google Container Registry).

**Authenticate with your Docker Registry:**

```bash
docker login

```

**Build and tag the image:**

> Replace `<your-username>` and `<your-app-name>` with your target registry details. The `.` at the end is required.

```bash
docker build -t <your-username>/<your-app-name>:latest .

```

**Push the image to the registry:**

```bash
docker push <your-username>/<your-app-name>:latest

```

```

```
