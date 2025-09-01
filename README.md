# Laravel Kubernetes Deployment

This repository contains Kubernetes manifests for deploying a complete Laravel PHP application stack with MySQL database and phpMyAdmin interface.

## Architecture Overview

The application follows a multi-container pod pattern with the following components:

```
┌─────────────────────────────────────────┐
│              laravel-app                │
│  ┌─────────────┐  ┌─────────────────┐   │
│  │    Nginx    │  │    PHP-FPM      │   │
│  │   (Alpine)  │  │   (Laravel)     │   │
│  │    :80      │  │    :9000        │   │
│  └─────────────┘  └─────────────────┘   │
└─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│              MySQL 8.0                 │
│            (Persistent)                │
│              :3306                     │
└─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│            phpMyAdmin                   │
│          (Web Interface)               │
│              :80                       │
└─────────────────────────────────────────┘
```

## Components

### 1. Laravel Application (`laravel-deployment.yaml`)

**Multi-container Pod Architecture:**
- **PHP-FPM Container**: Runs Laravel application code
  - Image: `ghcr.io/daveyrpt/flex-url:latest`
  - Port: 9000 (FastCGI)
- **Nginx Container**: Web server and reverse proxy
  - Image: `nginx:alpine`
  - Port: 80 (HTTP)
- **Init Container**: Sets up shared file permissions and copies application files

**Key Features:**
- Shared volume (`emptyDir`) for application files between containers
- Environment configuration via ConfigMap and Secrets
- Proper file permissions for Laravel storage and cache directories
- NodePort service exposing port 31119

### 2. MySQL Database (`mysql-deployment.yaml`)

**Components:**
- **Deployment**: Single replica MySQL 8.0 instance
- **PersistentVolumeClaim**: 5Gi storage for database persistence
- **Secret**: Base64-encoded credentials
- **Service**: ClusterIP service for internal access

**Credentials (Base64 encoded):**
- Root password: `rootpassword`
- Database: `laraveldb`
- User: `laraveluser`
- Password: `laravelpass`

### 3. phpMyAdmin (`phpmyadmin-deployment.yaml`)

**Features:**
- Web-based MySQL administration interface
- Connected to MySQL service automatically
- NodePort service exposing port 30080
- Uses MySQL root credentials from secrets

### 4. Configuration (`laravel-configmap.yaml`)

**Nginx Configuration:**
- PHP file processing via FastCGI
- Laravel-friendly URL rewriting
- Static file serving
- Security headers for `.ht` files

**Laravel Environment Variables:**
- Database connection settings
- Application configuration (debug mode enabled)
- Logging and caching configuration

## File Structure

```
.
├── laravel-deployment.yaml      # Laravel app with Nginx + PHP-FPM
├── laravel-configmap.yaml       # Environment variables + Nginx config
├── laravel-configmap.yaml.save  # Backup configuration
├── mysql-deployment.yaml        # MySQL database with persistence
└── phpmyadmin-deployment.yaml   # Database administration interface
```

## Deployment Instructions

### Prerequisites

- Kubernetes cluster (minikube, kind, or cloud provider)
- `kubectl` configured to access your cluster

### Deploy the Application

1. **Create namespace and deploy all components:**
   ```bash
   kubectl apply -f laravel-configmap.yaml
   kubectl apply -f mysql-deployment.yaml
   kubectl apply -f phpmyadmin-deployment.yaml
   kubectl apply -f laravel-deployment.yaml
   ```

2. **Verify deployment:**
   ```bash
   kubectl get pods -n laravel-app
   kubectl get services -n laravel-app
   ```

3. **Check logs if needed:**
   ```bash
   kubectl logs -n laravel-app deployment/laravel-app -c php-fpm
   kubectl logs -n laravel-app deployment/laravel-app -c nginx
   kubectl logs -n laravel-app deployment/mysql
   ```

### Access the Application

- **Laravel Application**: `http://localhost:31119` (or your node IP)
- **phpMyAdmin**: `http://localhost:30080` (or your node IP)

For cloud deployments, replace `localhost` with your node's external IP.

## Configuration Details

### Environment Variables

The Laravel application is configured through the `laravel-env` ConfigMap:

- **APP_ENV**: `local` (development mode)
- **APP_DEBUG**: `true` (error reporting enabled)
- **DB_HOST**: `mysql-service` (Kubernetes service name)
- **DB_DATABASE**: `laraveldb`
- **DB_USERNAME**: `laraveluser`

### Security Considerations

- Database passwords are stored in Kubernetes Secrets
- Laravel APP_KEY is properly configured
- Debug mode is enabled (disable for production)
- Default database credentials should be changed for production

### Storage

- **MySQL Data**: Persistent volume claim (5Gi)
- **Application Files**: Shared emptyDir volume between containers
- **Logs**: Container filesystem (consider persistent logging for production)

## Troubleshooting

### Common Issues

1. **Pods not starting**: Check resource availability and image pull status
2. **Database connection errors**: Verify MySQL service is running and secrets are correct
3. **File permission errors**: Check init container logs and volume mounts
4. **502 errors**: Verify PHP-FPM is running and Nginx configuration is correct

### Useful Commands

```bash
# Check pod status
kubectl get pods -n laravel-app -o wide

# View pod logs
kubectl logs -n laravel-app <pod-name> -c <container-name>

# Execute into containers
kubectl exec -it -n laravel-app <pod-name> -c php-fpm -- bash
kubectl exec -it -n laravel-app <pod-name> -c nginx -- sh

# Check services and endpoints
kubectl get svc,endpoints -n laravel-app
```

## Production Considerations

Before deploying to production, consider:

1. **Security**: Change default passwords, disable debug mode, use proper TLS
2. **Scalability**: Increase replicas, add horizontal pod autoscaling
3. **Persistence**: Use proper storage classes for MySQL data
4. **Monitoring**: Add health checks, logging, and monitoring solutions
5. **Backup**: Implement database backup strategies
6. **Resources**: Set appropriate CPU/memory requests and limits

## Development Workflow

1. Make changes to application code in your Laravel repository
2. Build and push new image to `ghcr.io/daveyrpt/flex-url:latest`
3. Update deployment to pull latest image:
   ```bash
   kubectl rollout restart deployment/laravel-app -n laravel-app
   ```
4. Monitor rollout:
   ```bash
   kubectl rollout status deployment/laravel-app -n laravel-app
   ```