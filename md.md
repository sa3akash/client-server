To ensure that the libraries are properly installed and built before building the Docker images for your Next.js client and Node.js server, you need to make sure that the client and server Dockerfiles are set up correctly to use the built libraries. 

Here's an updated workflow to install and build libraries, followed by building the server and client images, ensuring that libraries are available:

### Updated GitHub Actions Workflow

```yaml
name: Build and Push Docker Images

on:
  push:
    tags:
      - v*  # Trigger on version tags like v1.0.0

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Set up QEMU for multi-platform builds
      uses: docker/setup-qemu-action@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2

    - name: Install Node.js
      uses: actions/setup-node@v2
      with:
        node-version: '16'  # Specify your Node.js version

    - name: Install dependencies and build libraries
      run: |
        cd ./packages/libs  # Change to your libs directory
        npm install          # Install library dependencies
        npm run build        # Build the libraries

    - name: Login to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_TOKEN }}

    - name: Cache Docker layers
      uses: actions/cache@v3
      with:
        path: /tmp/.buildx-cache
        key: ${{ runner.os }}-buildx-${{ github.sha }}
        restore-keys: |
          ${{ runner.os }}-buildx-

    - name: Build and push Next.js Docker image
      uses: docker/build-push-action@v3
      with:
        context: ./packages/client  # Adjust to your Next.js app path
        file: ./packages/client/Dockerfile  # Adjust to your Dockerfile path
        build-args: |
          LIB_PATH=./libs  # Pass the library path as a build argument
        push: true
        tags: |
          sa2avroo/nextjs-standalone-new:latest
          sa2avroo/nextjs-standalone-new:${{ github.sha }}
          sa2avroo/nextjs-standalone-new:${{ github.ref_name }}

    - name: Build and push Node.js server Docker image
      uses: docker/build-push-action@v3
      with:
        context: ./packages/server  # Adjust to your Node.js server path
        file: ./packages/server/Dockerfile  # Adjust to your Dockerfile path
        build-args: |
          LIB_PATH=./libs  # Pass the library path as a build argument
        push: true
        tags: |
          sa2avroo/nodejs-server:latest
          sa2avroo/nodejs-server:${{ github.sha }}
          sa2avroo/nodejs-server:${{ github.ref_name }}

    - name: Logout from Docker Hub
      run: docker logout
```

### Key Adjustments:

1. **Library Installation**:
   - The step to install and build the libraries remains unchanged, ensuring the libraries are built before building the Docker images.

2. **Build Args**:
   - Added `build-args` to both the client and server build steps. This allows you to pass the library path to your Dockerfiles.

### Example Dockerfile for Client and Server

You need to adjust your `Dockerfile` for both the client and the server to use the `LIB_PATH` build argument. Here’s an example of what your Dockerfiles might look like:

#### Client Dockerfile (`./packages/client/Dockerfile`)

```dockerfile
# Use an appropriate base image
FROM node:16 AS build

# Set the working directory
WORKDIR /app

# Copy the package.json and package-lock.json
COPY package.json package-lock.json ./

# Install dependencies
RUN npm install

# Copy libraries built earlier
COPY ${LIB_PATH} ./libs  # Adjust the path as needed
COPY . .

# Build the Next.js app
RUN npm run build

# Start the app
CMD ["npm", "start"]
```

#### Server Dockerfile (`./packages/server/Dockerfile`)

```dockerfile
# Use an appropriate base image
FROM node:16 AS build

# Set the working directory
WORKDIR /app

# Copy the package.json and package-lock.json
COPY package.json package-lock.json ./

# Install dependencies
RUN npm install

# Copy libraries built earlier
COPY ${LIB_PATH} ./libs  # Adjust the path as needed
COPY . .

# Start the server
CMD ["npm", "start"]
```

### Important Notes:
- Ensure that the `LIB_PATH` in the Dockerfiles matches the directory structure where your libraries are built.
- You may need to adapt the `COPY` commands in the Dockerfiles depending on where the libraries are located after the build step.
- Make sure to test locally to confirm the library paths are set up correctly before pushing to your CI/CD pipeline.
- 


---


It looks like you've set up a GitHub Actions workflow to build and push Docker images. However, you mentioned that the action is not running and asked how to push a new release version. Here are some tips to help you troubleshoot and ensure that your action runs correctly when pushing a new release version:

### Key Points to Check:

1. **Triggering the Action**:
   - Your workflow is set to trigger on version tags (e.g., `v1.0.0`). Make sure you create a **tag** that follows your specified structure. For example:
     ```bash
     git tag v1.0.1
     git push origin v1.0.1
     ```
   - Ensure that there are no spaces or invalid characters in your tag name.

2. **Workflow Location**:
   - Make sure that your workflow YAML file is located in the correct directory, typically `.github/workflows/` in your repository.

3. **Branch Protection Rules**:
   - Check if there are any branch protection rules on the default branch that might prevent the GitHub Actions workflows from triggering.

4. **Permissions and Secrets**:
   - Ensure that the `DOCKER_USERNAME` and `DOCKER_TOKEN` secrets are correctly set in your repository settings. If the secrets are not set, the login step will fail, and the image will not be pushed.

5. **Check Your Actions Logs**:
   - After trying to push a new tag, go to the "Actions" tab in your GitHub repository to see if the action was triggered. If it was, you can check the logs for errors.

6. **Logs for Debugging**:
   - You may want to add a step to print out the current GitHub context to verify variables like `${{ github.sha }}` and `${{ github.ref_name }}` if you're facing issues.
   ```yaml
   - name: Print GitHub context
     run: echo "SHA: ${{ github.sha }}, Ref Name: ${{ github.ref_name }}"
   ```

7. **Using 'Latest' Tag**:
   - If you want to always push a new version with the `latest` tag, ensure that there’s a strategy in place to handle that, as pushing a `latest` tag every time might not be ideal if you're also tagging with version numbers.

### Example Steps to Create a New Release:

Assuming your current branch is `main`, here are the steps you can follow:

1. Create a new tag:
   ```bash
   git checkout main  # Make sure you're on the main branch
   git pull origin main  # Update your branch with the latest changes
   git tag v1.0.1  # Replace with your new version
   git push origin v1.0.1  # Push the tag to GitHub
   ```

2. After pushing the tag, navigate to the "Actions" tab in your repository to observe the build and push actions being executed.

### Final Note:
If you continue to face issues, you could consider enabling "Actions" for your repository if they are not already enabled, and review any logs or step outputs for helpful debugging information.

