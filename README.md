# Docker Deploy Action

A GitHub Action that builds, uploads, and deploys Docker images to a remote server via SSH.

## Features

- 🔨 **Automatic Docker Build**: Builds Docker images from your project's Dockerfile
- 📦 **Image Transfer**: Saves and transfers Docker images as tarballs to remote servers
- 🚀 **Remote Deployment**: Deploys containers on remote servers with automatic restart policies
- 🔧 **Environment Variables**: Supports custom environment variables for your containers
- 🔒 **SSH Authentication**: Secure deployment using SSH private keys
- 🏷️ **Version Management**: Organizes deployments by project name and build version

## Prerequisites

### On Your Remote Server

1. **Docker Installation**: Ensure Docker is installed and running on your target server
2. **SSH Access**: Set up SSH access with a dedicated user (default: `deployer`)
3. **Directory Permissions**: Ensure the SSH user can create directories in `/opt/`

### SSH User Setup (on your server)

```bash
# Create deployer user
sudo useradd -m -s /bin/bash deployer

# Add to docker group (for Docker commands)
sudo usermod -aG docker deployer

# Set up SSH key authentication
sudo mkdir -p /home/deployer/.ssh
sudo chown deployer:deployer /home/deployer/.ssh
sudo chmod 700 /home/deployer/.ssh

# Add your public key to authorized_keys
sudo -u deployer bash -c 'echo "YOUR_PUBLIC_KEY_HERE" >> /home/deployer/.ssh/authorized_keys'
sudo chmod 600 /home/deployer/.ssh/authorized_keys

# Set up /opt directory permissions for deployments
sudo mkdir -p /opt
sudo chown root:root /opt
sudo chmod 755 /opt
# The deployer user will create subdirectories as needed during deployment
```

## Usage

### Basic Usage

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Server
        uses: Armin-FalDiS/deployer-action@main
        with:
          server_address: 'your-server.com'
          ssh_private_key: ${{ secrets.SSH_PRIVATE_KEY }}
          port: '8080'
          env_vars: |
            NODE_ENV=production
            DATABASE_URL=${{ secrets.DATABASE_URL }}
```

### Advanced Usage

```yaml
name: Deploy with Custom Configuration

on:
  push:
    branches: [main, staging]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Server
        uses: Armin-FalDiS/deployer-action@main
        with:
          project_name: 'my-awesome-app'
          build_name: 'v1.2.3'
          server_address: 'production.example.com'
          ssh_private_key: ${{ secrets.SSH_PRIVATE_KEY }}
          ssh_user: 'deployer'
          port: '3000'
          container_port: '3000'
          env_vars: |
            NODE_ENV=production
            PORT=3000
            DATABASE_URL=${{ secrets.DATABASE_URL }}
            REDIS_URL=${{ secrets.REDIS_URL }}
            JWT_SECRET=${{ secrets.JWT_SECRET }}
          docker_args: '--network host --add-host host.docker.internal:host-gateway'
```

## Input Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `project_name` | No | Repository name | Name of the project (used for container naming) |
| `build_name` | No | Branch name | Build name/version identifier |
| `server_address` | Yes | - | IP address or hostname of your deployment server |
| `ssh_private_key` | Yes | - | SSH private key for server authentication |
| `ssh_user` | No | `deployer` | SSH username for server access |
| `port` | No | `3000` | Host port to expose the container on |
| `container_port` | No | `3000` | Container's internal port |
| `env_vars` | No | - | Environment variables in `KEY=value` format (one per line) |
| `docker_args` | No | - | Additional Docker run arguments |

## GitHub Secrets Setup

You'll need to set up these secrets in your repository:

1. **SSH_PRIVATE_KEY**: Your SSH private key for server access
   ```bash
   # Generate a new SSH key pair
   ssh-keygen -t ed25519 -C "github-actions@your-domain.com"
   
   # Copy the private key content to GitHub Secrets
   cat ~/.ssh/id_ed25519
   ```

2. **Other Environment Variables**: Any sensitive environment variables your application needs

## How It Works

1. **Build**: Creates a Docker image from your project's Dockerfile
2. **Package**: Saves the image as a tarball for efficient transfer
3. **Transfer**: Uploads the image and environment file to the remote server
4. **Deploy**: Loads the image and runs it as a container with restart policy
5. **Cleanup**: Removes old containers and prunes unused images

## Project Structure

Your project should have a `Dockerfile` in the root directory:

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

## Deployment Locations

The action deploys to:
- **Container Name**: `{project_name}-{build_name}`
- **Local Directory**: `/opt/{project_name}/{build_name}/`
- **Container Port**: `127.0.0.1:{port}:{container_port}`

## Troubleshooting

### Common Issues

1. **SSH Connection Failed**
   - Verify your SSH private key is correct
   - Ensure the server is accessible from GitHub Actions
   - Check SSH user permissions

2. **Docker Permission Denied**
   - Ensure the SSH user is in the `docker` group
   - Restart the SSH session after adding to docker group

3. **Port Already in Use**
   - The action automatically removes old containers with the same name
   - Check for other services using the same port

4. **Environment Variables Not Working**
   - Ensure the format is `KEY=value` (one per line)
   - Check that your application reads from the `.env` file

### Debug Mode

To debug deployment issues, you can SSH into your server and check:

```bash
# Check running containers
docker ps

# Check container logs
docker logs {project_name}-{build_name}

# Check if image was loaded
docker images | grep {project_name}

# Check deployment directory
ls -la /opt/{project_name}/{build_name}/
```

## Security Considerations

- Use dedicated SSH keys for GitHub Actions
- Restrict SSH user permissions to only what's necessary
- Use secrets for sensitive environment variables
- Consider using a reverse proxy (like nginx) for production deployments
- Regularly rotate SSH keys and secrets

## License

This action is provided as-is for deployment automation. Use at your own risk in production environments.
