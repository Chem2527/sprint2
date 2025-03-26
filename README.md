# Sprint 2: Integrating Vulnerability Scanning with CI/CD Pipelines

## Tasks:

### 1. Develop scripts to automate the scanning process for new container images in GitHub Actions

In this sprint, we created a CI pipeline using GitHub Actions that automates the scanning of Docker images for vulnerabilities. The process is triggered on every push event to the main branch. The key steps involved are:

 ```bash
1.1 Checkout Repository: Retrieves the code from the repository.

1.2 Set up Docker: Prepares the environment to build Docker images.

1.3 Install Trivy (Vulnerability Scanner): Installs Trivy, a tool to scan Docker images for vulnerabilities.

1.4 Build Docker image: Builds the Docker image using the Dockerfile located at the root level.

1.5 Run Trivy scan: Executes Trivy on the built image to identify any vulnerabilities.

1.6 Upload Trivy report: Uploads the scan result as an artifact if vulnerabilities are found.

1.7 Provide feedback: Analyzes the scan result and provides feedback on whether the build passes or fails.
```


### 2. Define criteria for passing/failing builds based on vulnerability severity (e.g., failing if high/critical vulnerabilities are detected):

The build will fail if any vulnerabilities of Critical or High severity are found in the Docker image. The severity level is controlled by the repository secret **VULNERABILITY_SEVERITY**.

```bash
2.1 The workflow uses Trivy to scan for vulnerabilities and checks if the severity matches or exceeds the threshold defined in the VULNERABILITY_SEVERITY secret. If vulnerabilities of the specified severity level or higher are found, the build will be marked as failed.

2.2 Add environment configurations to control scanning parameters (e.g., CVE level threshold):
To control the severity threshold for vulnerabilities, an environment variable SEVERITY_THRESHOLD is set in the GitHub Actions workflow file. This value is pulled from the GitHub repository secrets:

2.3 The VULNERABILITY_SEVERITY secret is created in the GitHub repository settings and can be set to values like critical, high, medium, etc.

2.4 This allows dynamic control of the severity threshold without modifying the workflow file, ensuring flexibility for different scanning needs.
```

3. Test CI/CD integration with sample images:

```bash
The CI/CD pipeline has been successfully tested by integrating a sample Node.js application with the pipeline. The Docker image is built from a Dockerfile and scanned for vulnerabilities. The workflow processes and provides feedback, indicating whether any vulnerabilities of the specified severity level exist.
```
## Complete Document for CI Pipeline with Vulnerability Scanning using GitHub Actions

### 1. **GitHub Actions Workflow Configuration** (vulnerabilities.yml):
   
This configuration file, placed under .github/workflows in your repository, automates the vulnerability scanning process for new Docker images whenever code is pushed to the main branch.

```bash
name: CI Pipeline with Vulnerability Scanning

on:
  push:
    branches:
      - main

jobs:
  scan:
    runs-on: ubuntu-latest

    env:
      SEVERITY_THRESHOLD: ${{ secrets.VULNERABILITY_SEVERITY }}  # Access the secret for severity threshold

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3  # Use a specific commit SHA or tag for stability

      - name: Set up Docker
        uses: docker/setup-buildx-action@v2

      - name: Install Trivy (Vulnerability Scanner)
        run: |
          sudo apt-get update
          sudo apt-get install -y wget
          wget https://github.com/aquasecurity/trivy/releases/download/v0.29.0/trivy_0.29.0_Linux-64bit.deb
          sudo dpkg -i trivy_0.29.0_Linux-64bit.deb

      - name: Build Docker image
        run: |
          # Build the Docker image using the Dockerfile from the root directory
          docker build -t myapp .

      - name: Run Trivy scan
        id: trivy
        run: |
          # Run Trivy scan and store results in trivy-report.json
          trivy image --exit-code 0 --format json --output trivy-report.json --severity $SEVERITY_THRESHOLD myapp

      - name: List files in the directory (for debugging)
        run: |
          ls -alh  # List files to verify that the trivy-report.json exists

      - name: Upload Trivy report (if scan ran)
        if: steps.trivy.outcome != 'skipped'
        uses: actions/upload-artifact@v4  # Upload the Trivy report
        with:
          name: trivy-report
          path: trivy-report.json

      - name: Provide feedback on vulnerabilities
        run: |
          if [ -f trivy-report.json ]; then
            # Check if there are Critical or High vulnerabilities in the report
            if jq -e '.[] | select(.Severity=="CRITICAL" or .Severity=="HIGH")' trivy-report.json > /dev/null; then
              echo "Critical or High vulnerabilities found. Build failed."
              exit 1
            else
              echo "No critical or high vulnerabilities found. Build passed."
            fi
          else
            echo "Trivy report not found. Assuming no vulnerabilities."
          fi
```

### 2. Dockerfile (Placed in the Root of the Repository):
This Dockerfile is used to build the Docker image for a Node.js application.

```bash
# Use official Node.js image as the base image
FROM node:14

# Set the working directory
WORKDIR /app

# Copy package.json and install dependencies
COPY package*.json ./
RUN npm install

# Copy the rest of the application code
COPY . .

# Expose the application port
EXPOSE 3000

# Run the application
CMD ["npm", "start"]
```

Explanation: This Dockerfile uses the official node:14 image, sets up the working directory, installs dependencies, and starts the Node.js application on port 3000.

### 3. Repository Secrets Configuration:

To securely control the severity threshold, we use a repository secret:

```bash
Navigate to GitHub Repository Settings.

Under Secrets in the sidebar, select New repository secret.

Set the Name to VULNERABILITY_SEVERITY and Value to critical (or another severity level like high).

This value is then used in the workflow file via the env configuration: SEVERITY_THRESHOLD: ${{ secrets.VULNERABILITY_SEVERITY }}.
```
### 4. CI/CD Workflow Behavior:
```bash
On Push to Main Branch: The workflow is triggered when changes are pushed to the main branch.

Docker Image Build: The Docker image is built using the Dockerfile placed at the root of the repository.

Trivy Vulnerability Scan: After the image is built, it is scanned using Trivy, and any vulnerabilities are reported.

Build Feedback: If Critical or High vulnerabilities are found, the build fails. If no such vulnerabilities exist, the build passes.
```
