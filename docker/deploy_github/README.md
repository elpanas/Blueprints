# AutoCheckDeploy

This repository contains an **automated deployment script** for containerized applications using Docker Compose. The script updates the application on a remote server via SSH, with automatic backups and the possibility of rollback in case of errors.

---

## General Workflow

The workflow is implemented as a GitHub Action (`.github/workflows/deploy.yml`) and is triggered when pushes are made to the configured branches (`main` or `dev` by default).  

The deployment process follows these main steps:

1. **Checkout code**  
   The Action downloads the deployment files from the repository to the GitHub runner using `actions/checkout@v4`.

2. **Load configuration**  
   - Reads the `deploy.conf` file located in the **root** of the repository.  
   - Exports the main variables (`APP_NAME`, `APP_PATH`, `SERVICES`, `GIT_REPO`, `VOLUMES`, `HEALTHCHECK_TIMEOUT`) as environment variables for the following steps.

3. **SSH connection to the server**  
   - Uses the `appleboy/ssh-action@v0.1.10` action to connect to the server via SSH.  
   - Credentials are retrieved from GitHub Secrets: `SERVER_HOST`, `SERVER_USERNAME`, `SERVER_SSH_KEY`.

4. **Initialization and preparation**  
   - Defines the branch to deploy (`main`, `dev`, or any other listed branch).  
   - Converts the `SERVICES` and `VOLUMES` strings into lists.  
   - Creates log and backup directories (`logs/` and `backups/`).  
   - Prepares a timestamped log file.

5. **Volume backups**  
   - Each volume listed in `VOLUMES` is saved as a `.tar.gz` file.  
   - Backups are written to the `backups/` folder on the server.

6. **Docker image backup for rollback**  
   - Each service image is cloned as a rollback image (`<SERVICE>_rollback`).  
   - Allows restoring the previous version in case of errors.

7. **Repository update**  
   - If the folder does not contain `.git`, it clones the repository from the correct branch.  
   - If the folder already contains the repo, it performs `fetch` and `reset --hard` on the target branch to avoid conflicts.

8. **Stop existing containers**  
   - Executes `docker compose down` to stop active containers and prepare for the new deployment.

9. **Build and start new containers**  
   - Runs `docker compose up --build -d`.  
   - For each service:
     - Checks the number of restarts and the health status (`healthcheck`).  
     - If there is an error, executes **automatic rollback**.  
     - Shows the last 50 logs of the container.

10. **Deployment completed**  
    - Confirms deployment if all services start successfully.  
    - Rollback images are preserved for potential future use.

---

## Main Variables (`deploy.conf`)

| Variable             | Description | Example |
|----------------------|-------------|---------------------------|
| `APP_NAME`           | Name of the application | your-app |
| `APP_PATH`           | Path on the server where the project is located | /srv/your-app |
| `SERVICES`           | List of Docker Compose services, comma-separated | backend,frontend |
| `GIT_REPO`           | SSH URL of the repository | git@github.com:username/repo.git |
| `VOLUMES`            | List of Docker volumes to backup, comma-separated | volume1,volume2 |
| `HEALTHCHECK_TIMEOUT`| Timeout in seconds for container health checks | 60 |

---

## Automatic Rollback

If a container fails the healthcheck or restarts multiple times, the script:

1. Shows the last logs of the failed container.  
2. Saves the full logs to a timestamped file.  
3. Restores the previous image.  
4. Restarts the containers with the working version.  
5. Exits with code 1, marking the deployment as failed.
