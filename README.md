# Goal

Practice:
* Containerize 2 services together.
* CI/CD pipeline by using Github Actions
* Deploying on AWS EC2

___

## Project Structure

We will be using Redis and Nginx to get conainerized and deployed.

```
nginx-redis-project/
├── docker-compose.yml
├── nginx/
│   ├── nginx.conf
│   └── html/
│       └── index.html
├── .github/
│   └── workflows/
│       └── deploy.yml
└── README.md
```

## Steps

### Phase 1

1) In `index.html` write something to display when the website gets deployed.

2) Create a custom `nginx.conf` to override the default configuration provided by the NGINX image.

3) Write `docker-compose.yml` which helps to containerize multiple services or docker images together. 
    
    No need to mount volume for Ndinx here, As it's not ideal when:

    * You want to ship a complete Docker image to production (e.g., push to DockerHub or AWS ECR)

    * You want to scan the image with Trivy

    * You want to use distroless or Alpine for security

    * You want to remove volume dependencies (volume mounting won’t work easily in ECS or EC2 at scale)

4) Instead create `Dockerfile.nginx` as this approach bakes your content + config directly into the container image. 

    This gives you:

    🔒 Security

    * Avoids dynamic mounts

    * Easier to scan for vulnerabilities with Trivy

    * Can later switch to distroless or non-root base

    📦 Portability

    * Image works anywhere without needing html/ or nginx.conf on disk

    * Great for pushing to DockerHub or deploying on EC2/ECS/Kubernetes

    🤖 CI/CD Friendly

    * GitHub Actions can build & push the image

    * No dependency on local files during deployment


### Phase 2 

5) Create `deploy.yml` to ci/cd functions.

    ***Jobs Sequence:***
    
    1) A single job that runs on a GitHub-hosted runner using Ubuntu 24.04.All your CI/CD logic is wrapped under this job.<br><br> 
    
    ```yaml  
    jobs:
        build-and-scan:
            runs-on: ubuntu-24.04
    ```

    2) **Set up Docker Buildx:** Enables advanced Docker builds, including multi-platform and cache support. Necessary for using build-push-action.

    3) **Login to DockerHub:** Authenticates with Docker Hub using GitHub secrets. Allows GitHub Actions to push the built image to your Docker Hub account.

        * Go to Github Repository -> Select your Repo -> Settings -> Secrets & Variables -> Actions 
            
    4) **Build and Push Image:** Builds the Docker image using Dockerfile.nginx. Pushes it to Docker Hub under the custom-nginx:latest tag. The image includes all HTML and NGINX config as per your Dockerfile.

    5) **Security Scan with Trivy:** Scans the built image for vulnerabilities. exit-code: 0 means it won’t fail the workflow even if issues are found, just reports them.
    We are just practicing so we won't stop the process to fix vulnerabilities. 

    6) **Setup SSH:** Prepares SSH credentials to access your EC2 instance.EC2_SSH_KEY: the private key in base64 or plain string format, stored securely in GitHub Secrets.

    7) **Deploy via SSH:** Connects to your EC2 instance and performs:

        * Pull latest image from Docker Hub

        * Stop and remove old container (if it exists)

        * Run new container with the updated image on port 80
    
    #### ✅ Summary Flow:
<!-- 
    ```plaintext
    📦 Push to main
    ⮑ ✅ IF: not README or .md file
    ⮑ Docker image is built from Dockerfile.nginx
    ⮑ Image is pushed to Docker Hub
    ⮑ Trivy scans for vulnerabilities
    ⮑ SSH into EC2 and deploys the image
    ``` -->
### Phase 3

7) Generate SSH Key (for GitHub Actions to access EC2)

```bash
ssh-keygen -t rsa -b 4096 -C "github-deploy" -f ~/.ssh/github-actions-key`
```

* It generates 2 files:
    * github-actions-key (private)
    * github-actions-key.pub (public)

8) Add Public Key to EC2 in `~/.ssh/authorized_keys`

9) Add Private Key to GitHub Secrets

## Outcome

Now whenever you push to main branch of the reposiotry Github actions will trigger, and your repo image will be built and deployed automatically.

* Every push to the main branch will:

    * Build your NGINX image using your Dockerfile

    * Push the image to Docker Hub

    * Scan it for security issues

    * SSH into your EC2 instance

    * Stop the old container and run the new one
