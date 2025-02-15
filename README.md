# Weather App MySQL Database Setup for Kubernetes

This project provides Kubernetes configurations for setting up a MySQL database for a weather application. It includes a StatefulSet for persistent storage, an initialization job, and a headless service for database discovery.

The MySQL database is designed to support the authentication component of the weather application. It ensures data persistence, proper initialization of the database and user, and provides a discoverable endpoint for other services within the Kubernetes cluster.

## Repository Structure

```
weather-app/
└── auth/
    └── mysql/
        ├── headless-service.yaml
        ├── init-job.yaml
        └── statefulset.yaml
```

- `headless-service.yaml`: Defines a Kubernetes Service for MySQL database discovery.
- `init-job.yaml`: Contains a Kubernetes Job for initializing the MySQL database.
- `statefulset.yaml`: Specifies a Kubernetes StatefulSet for running the MySQL database with persistent storage.

## Usage Instructions

### Prerequisites

- Kubernetes cluster (version 1.19+)
- `kubectl` command-line tool (version 1.19+)
- Kubernetes Secret named `mysql-secret` containing:
  - `root-password`: MySQL root password
  - `auth-password`: Password for the `weatherapp` user
  - `secret-key`: To use with JWT

### Deployment

1. Create the headless service:

```bash
kubectl apply -f weather-app/auth/mysql/headless-service.yaml
```

2. Apply the StatefulSet:

```bash
kubectl apply -f weather-app/auth/mysql/statefulset.yaml
```

3. Run the initialization job:

```bash
kubectl apply -f weather-app/auth/mysql/init-job.yaml
```

### Verification

To verify the deployment:

1. Check if the StatefulSet is running:

```bash
kubectl get statefulset mysql
```

2. Verify the headless service:

```bash
kubectl get service mysql
```

3. Check the status of the initialization job:

```bash
kubectl get job mysql-init-job
```

### Connecting to the Database

Other services within the Kubernetes cluster can connect to the MySQL database using the following connection details:

- Host: `mysql.default.svc.cluster.local` (assuming the default namespace)
- Port: 3306
- User: weatherapp
- Password: Retrieved from the `mysql-secret` Kubernetes Secret
- Database: weatherapp

### Troubleshooting

1. If the StatefulSet fails to start:
   - Check the pod events: `kubectl describe pod mysql-0`
   - View the pod logs: `kubectl logs mysql-0`

2. If the initialization job fails:
   - Check the job events: `kubectl describe job mysql-init-job`
   - View the job logs: `kubectl logs job/mysql-init-job`

3. If services can't connect to the database:
   - Ensure the headless service is running: `kubectl get service mysql`
   - Verify DNS resolution within the cluster: `kubectl run -it --rm --restart=Never mysql-client --image=mysql:5.7 -- mysql -h mysql -pYOUR_ROOT_PASSWORD`

## Data Flow

1. The StatefulSet creates a MySQL pod with persistent storage.
2. The headless service provides a stable network identity for the MySQL pod.
3. The initialization job connects to the MySQL instance and sets up the required database and user.
4. Other services in the cluster can then connect to the MySQL database using the provided credentials and the headless service DNS name.

```
[StatefulSet] --> (Creates) --> [MySQL Pod]
[Headless Service] --> (Provides DNS) --> [MySQL Pod]
[Initialization Job] --> (Configures) --> [MySQL Pod]
[Other Services] --> (Connect via) --> [Headless Service] --> [MySQL Pod]
```

## Infrastructure

### StatefulSet

- **Type**: `apps/v1.StatefulSet`
- **Name**: mysql
- **Purpose**: Manages the deployment and scaling of the MySQL database pod

Key configurations:
- Replica count: 1
- Container image: mysql:latest
- Root password: Retrieved from `mysql-secret` Kubernetes Secret
- Persistent volume claim: 10Gi storage using the "standard" storage class

### Job

- **Type**: `batch/v1.Job`
- **Name**: mysql-init-job
- **Purpose**: Initializes the MySQL database with required schema and user

Key operations:
- Creates "weatherapp" database
- Creates "weatherapp" user with password from `mysql-secret`
- Grants all privileges on "weatherapp" database to "weatherapp" user

### Service

- **Type**: `v1.Service`
- **Name**: mysql
- **Purpose**: Provides a stable network endpoint for MySQL database discovery

Key configurations:
- Headless service (clusterIP: None)
- Exposes port 3306
- Selects pods with label `app: mysql`

## Additional Components

The weather application consists of several components beyond the MySQL database:
- UI: A user interface component exposed via an Ingress
- Auth: An authentication service
- Weather: A weather data service

Each of these components has its own Deployment and Service configurations.