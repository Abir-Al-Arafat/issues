# 🛠️ Troubleshooting Guide: MongoDB Replica Set Connection in Docker

This document explains the root cause and the permanent fix for the "Server selection timeout" or "Connection refused" errors encountered when running a Node.js/Prisma application inside a Docker container while connecting to a MongoDB Replica Set on an EC2 host.

---

## 1. The Problem: "Connection Refused" at 127.0.0.1

### Symptoms

Even after updating the `DATABASE_URL` in the `.env` file to use the EC2 private IP, the application logs continue to show an error like this:
`Kind: I/O error: Connection refused (os error 111) ... Servers: [ { Address: 127.0.0.1:27017 } ]`

### Root Cause

In a Docker environment, `127.0.0.1` refers to the container itself. When your application connects to a MongoDB Replica Set:

1.  The app initially connects to the seed IP provided in your environment variables (e.g., `172.31.4.74`).
2.  MongoDB responds with its internal **Replica Set Configuration**.
3.  If that internal configuration identifies the primary node as `localhost` or `127.0.0.1`, the database tells the application: "I am actually at 127.0.0.1".
4.  The application then attempts to switch its connection to `127.0.0.1` **inside the container**, where no database exists, leading to a crash.

---

## 2. The Solution: Reconfiguring the Replica Set

You must force MongoDB to identify itself by its actual network IP so that external clients (like Docker containers) can find it.

### Step 1: Update MongoDB Internal Configuration

1.  Log into your EC2 host terminal and enter the MongoDB shell:
    ```bash
    mongosh
    ```
2.  Run the following commands inside the shell to swap the internal hostname (replace `172.31.4.74` with your current private IP):

    ```javascript
    // 1. Get the current configuration
    cfg = rs.conf();

    // 2. Replace '127.0.0.1' with your EC2 Private IP
    cfg.members[0].host = "172.31.4.74:27017";

    // 3. Apply the new configuration
    rs.reconfig(cfg);
    ```

### Step 2: Update MongoDB Network Binding

Ensure MongoDB is listening on the private network interface, not just the local loopback.

1.  Open the MongoDB configuration file:
    ```bash
    sudo nano /etc/mongod.conf
    ```
2.  Update the `net` section to include your private IP:
    ```yaml
    net:
      port: 27017
      bindIp: 127.0.0.1,172.31.4.74
    ```
3.  Restart MongoDB to apply changes:
    ```bash
    sudo systemctl restart mongod
    ```

### Step 3: Restart the Docker Container

Ensure your `.env` file reflects the correct connection string:
`DATABASE_URL="mongodb://172.31.4.74:27017/borla_db?replicaSet=rs0"`

Force a clean restart of the container:

```bash
docker stop borla-backend-api && docker rm borla-backend-api
docker run -d \
  --name borla-backend-api \
  --restart always \
  -p 5000:5000 \
  --env-file ~/borla_backend/.env \
  abirwerks/borla-backend:latest
```

---

## 3. Summary Checklist

- **Binding:** MongoDB is bound to the EC2 Private IP in `mongod.conf`.
- **Replica Set:** The member host is set to the EC2 Private IP in `rs.conf()` (check using `rs.status()`).
- **Environment:** The container is launched using an `--env-file` that points to the correct network IP.
