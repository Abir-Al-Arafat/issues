# Guide: EC2 DISK SPACE ISSUE

This guide outlines the step-by-step process for identifying disk space issues, expanding EBS storage on AWS, and setting up the environment for deploying React/Next.js applications.

## Part 1: Identifying and Resolving Disk Space Issues

When building Node.js applications, dependencies and build processes require significant disk space. If you encounter `ENOSPC: no space left on device` errors, your instance volume is full.

1.  **Check Disk Usage:**
    Use the command `df -h` to check the current utilization of your partitions.
2.  **Verify Volume Details:**
    In the AWS EC2 Console, navigate to the **Instances** page, select your instance, and go to the **Storage** tab to identify the volume ID.
3.  **Expand EBS Volume:**
    * Navigate to **Elastic Block Store > Volumes**.
    * Select your volume, go to **Actions > Modify volume**.
    * Increase the **Size (GiB)** and click **Modify**.

## Part 2: Resizing the File System

AWS increases the physical volume, but the OS needs to be notified to use the new capacity.

1.  **Detect Changes:**
    Run `lsblk` to confirm the OS detects the new disk size (e.g., 20G).
2.  **Grow the Partition:**
    Use `growpart` to expand the partition. For the root filesystem:
    `sudo growpart /dev/nvme0n1 1`
3.  **Resize File System:**
    Update the filesystem to recognize the expanded space:
    `sudo resize2fs /dev/nvme0n1p1`

## Part 3: Environment Setup (NVM & Node.js)

Your project likely requires a newer Node.js version than what is installed by default.

1.  **Install NVM (Node Version Manager):**
    Use the official installation script:
    `curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash`
2.  **Load NVM:**
    Refresh your shell or run:
    `export NVM_DIR="$HOME/.nvm" && [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"`
3.  **Install Node.js:**
    `nvm install 20`
    `nvm use 20`

## Part 4: Final Deployment

With disk space and the correct Node environment, you can now safely install your project.

1.  **Cleanup:** Remove old files that might be corrupted or bloated:
    `rm -rf node_modules package-lock.json`
2.  **Install Dependencies:**
    `npm install`
3.  **Build (Recommended):**
    For production, it is best practice to build on your local machine and only transfer the build artifacts, but if building on the server:
    `npm run build`
