# Laravel Helm Chart

This Helm chart deploys a complete Laravel application stack with MySQL database and phpMyAdmin on Kubernetes.

## Prerequisites

- Kubernetes cluster (1.19+)
- Helm 3.0+
- kubectl configured to access your cluster

## Installation

### Quick Start

1. **Install the chart with default values:**
   ```bash
   helm install laravel-app ./helm/laravel-app
   ```

2. **Install with custom values:**
   ```bash
   helm install laravel-app ./helm/laravel-app -f custom-values.yaml
   ```

3. **Install in a specific namespace:**
   ```bash
   helm install laravel-app ./helm/laravel-app --namespace my-laravel --create-namespace
   ```

## Configuration

### Key Configuration Options

| Parameter | Description | Default |
|-----------|-------------|---------|
| `global.namespace` | Kubernetes namespace | `laravel-app` |
| `laravel.image.repository` | Laravel image repository | `ghcr.io/daveyrpt/flex-url` |
| `laravel.image.tag` | Laravel image tag | `latest` |
| `laravel.replicas` | Number of Laravel replicas | `1` |
| `laravel.service.nodePort` | NodePort for Laravel service | `31119` |
| `mysql.enabled` | Enable MySQL deployment | `true` |
| `mysql.auth.rootPassword` | MySQL root password | `rootpassword` |
| `mysql.auth.database` | MySQL database name | `laraveldb` |
| `mysql.auth.username` | MySQL username | `laraveluser` |
| `mysql.auth.password` | MySQL password | `laravelpass` |
| `mysql.persistence.enabled` | Enable persistent storage | `true` |
| `mysql.persistence.size` | PVC size for MySQL | `5Gi` |
| `phpmyadmin.enabled` | Enable phpMyAdmin | `true` |
| `phpmyadmin.service.nodePort` | NodePort for phpMyAdmin | `30080` |

### Custom Values Example

Create a `custom-values.yaml` file:

```yaml
laravel:
  replicas: 2
  image:
    tag: "v1.2.3"
  env:
    APP_ENV: "production"
    APP_DEBUG: "false"
  service:
    nodePort: 32000

mysql:
  auth:
    rootPassword: "my-secure-password"
    database: "my-laravel-db"
    username: "my-user"
    password: "my-password"
  persistence:
    size: 10Gi

resources:
  laravel:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi
```

## Deployment Commands

### Install/Upgrade
```bash
# Install new release
helm install laravel-app ./helm/laravel-app

# Upgrade existing release
helm upgrade laravel-app ./helm/laravel-app

# Install or upgrade (create if doesn't exist)
helm upgrade --install laravel-app ./helm/laravel-app
```

### Manage Releases
```bash
# List releases
helm list

# Get release status
helm status laravel-app

# Get release values
helm get values laravel-app

# Uninstall release
helm uninstall laravel-app
```

### Debugging
```bash
# Dry run to see generated manifests
helm install laravel-app ./helm/laravel-app --dry-run --debug

# Template without installing
helm template laravel-app ./helm/laravel-app

# Validate chart
helm lint ./helm/laravel-app
```

## Accessing the Application

After installation, the chart will provide URLs in the NOTES output. Typically:

- **Laravel Application**: `http://<node-ip>:31119`
- **phpMyAdmin**: `http://<node-ip>:30080`

Get your node IP:
```bash
kubectl get nodes -o wide
```

## Customization Examples

### Disable phpMyAdmin
```yaml
phpmyadmin:
  enabled: false
```

### Use LoadBalancer Service
```yaml
laravel:
  service:
    type: LoadBalancer
    # nodePort not needed for LoadBalancer

phpmyadmin:
  service:
    type: LoadBalancer
```

### External MySQL
```yaml
mysql:
  enabled: false

laravel:
  env:
    DB_HOST: "external-mysql.example.com"
    DB_PORT: "3306"
```

### Production Configuration
```yaml
laravel:
  replicas: 3
  env:
    APP_ENV: "production"
    APP_DEBUG: "false"
    LOG_LEVEL: "warning"

mysql:
  persistence:
    size: 50Gi
    storageClass: "fast-ssd"

resources:
  laravel:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 2000m
      memory: 2Gi
  mysql:
    requests:
      cpu: 1000m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 4Gi
```

## Troubleshooting

### Common Issues

1. **Image Pull Errors**
   ```bash
   kubectl describe pod -n laravel-app <pod-name>
   ```

2. **Database Connection Issues**
   ```bash
   kubectl logs -n laravel-app deployment/laravel-app -c php-fpm
   kubectl exec -it -n laravel-app deployment/mysql -- mysql -u root -p
   ```

3. **Service Access Issues**
   ```bash
   kubectl get svc -n laravel-app
   kubectl describe svc -n laravel-app laravel-service
   ```

### Useful Commands
```bash
# Check all resources
kubectl get all -n laravel-app

# Check persistent volumes
kubectl get pv,pvc -n laravel-app

# Port forward for local access
kubectl port-forward -n laravel-app svc/laravel-service 8080:80
kubectl port-forward -n laravel-app svc/phpmyadmin-service 8081:80
```

## Chart Structure

```
helm/laravel-app/
├── Chart.yaml                      # Chart metadata
├── values.yaml                     # Default configuration values
├── templates/
│   ├── _helpers.tpl                # Template helpers
│   ├── NOTES.txt                   # Post-install notes
│   ├── namespace.yaml              # Namespace resource
│   ├── laravel-configmap.yaml      # Laravel configuration
│   ├── laravel-secrets.yaml        # Laravel secrets
│   ├── laravel-deployment.yaml     # Laravel deployment and service
│   ├── mysql-secrets.yaml          # MySQL secrets
│   ├── mysql-pvc.yaml              # MySQL persistent volume claim
│   ├── mysql-deployment.yaml       # MySQL deployment and service
│   └── phpmyadmin-deployment.yaml  # phpMyAdmin deployment and service
└── .helmignore                     # Files to ignore when packaging
```

## Contributing

1. Make changes to the chart
2. Test with `helm lint ./helm/laravel-app`
3. Test installation with `helm install --dry-run --debug`
4. Update version in `Chart.yaml`
5. Update documentation