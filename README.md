# PostgresDatabase CRD

A developer-friendly Kubernetes CRD that simplifies PostgreSQL database creation by abstracting away the complexity of Percona's PerconaPGCluster operator.

## Overview

Instead of writing 50+ lines of complex YAML, developers can create a PostgreSQL database with just 8 lines:

```yaml
apiVersion: databases.mycompany.com/v1
kind: PostgresDatabase
metadata:
  name: billing-db
  namespace: production
spec:
  version: 17
  replicas: 3
  storage: 100Gi
  backup: true
  monitoring: true
```

## Features

- **Simple API**: Only essential fields exposed to developers
- **Version Support**: PostgreSQL 13, 14, 15, 16, 17
- **High Availability**: Automatic HA configuration with 3+ replicas
- **Built-in Backups**: Automated backups with configurable retention
- **Monitoring**: PMM integration by default
- **Resource Management**: Sensible CPU/memory defaults
- **Validation**: OpenAPI v3 schema validation prevents configuration errors

## Installation

### 1. Install the CRD

```bash
kubectl apply -f deploy/crd-postgres-database.yaml
```

### 2. Deploy the Controller

See [postgres-database-controller](https://github.com/AgnosticDBA/postgres-database-controller) for controller deployment instructions.

## Usage

### Create a Database

```bash
kubectl apply -f deploy/test-postgres-database.yaml
```

### Monitor Creation

```bash
kubectl get postgresdatabases -w
```

### Connect to Database

```bash
kubectl port-forward svc/billing-db 5432:5432
psql -h localhost -U postgres -d billing_db
```

## API Reference

### Spec Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `version` | integer (13-17) | Required | PostgreSQL major version |
| `replicas` | integer (1-10) | 1 | Number of replicas (3+ for HA) |
| `storage` | string (e.g., "100Gi") | Required | Storage capacity |
| `backup` | boolean | true | Enable automated backups |
| `monitoring` | boolean | true | Enable PMM monitoring |
| `backupRetention` | string | "7d" | Backup retention period |
| `resources.requests.cpu` | string | "100m" | CPU request |
| `resources.requests.memory` | string | "256Mi" | Memory request |
| `resources.limits.cpu` | string | "500m" | CPU limit |
| `resources.limits.memory` | string | "1Gi" | Memory limit |

### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `phase` | string | Current state (Pending, Creating, Ready, Failed) |
| `endpoint` | string | Connection endpoint |
| `replicas` | integer | Actual ready replica count |
| `message` | string | Status message or error details |
| `credentialsSecret` | string | Secret containing connection credentials |
| `lastBackupTime` | timestamp | Last successful backup time |

## Architecture

```
┌─────────────────┐    ┌─────────────────────┐    ┌──────────────────┐
│ PostgresDatabase │───▶│ Platform Controller │───▶│ PerconaPGCluster │
│ (8 lines YAML)  │    │ (Abstracts complexity)│    │ (50+ lines YAML) │
└─────────────────┘    └─────────────────────┘    └──────────────────┘
```

## Development

### Test Examples

The `deploy/test-postgres-database.yaml` contains 4 test scenarios:

1. **Production HA Database** - 3 replicas, backups, monitoring
2. **Development Database** - 1 replica, minimal resources
3. **High-Performance Database** - 5 replicas, large storage
4. **Legacy Compatibility** - PostgreSQL 15 for older applications

### Validation

```bash
kubectl apply --dry-run=client -f deploy/crd-postgres-database.yaml
kubectl apply --dry-run=client -f deploy/test-postgres-database.yaml
```

## Repository Structure

```
postgres-database/
├── deploy/
│   ├── crd-postgres-database.yaml      # CRD definition
│   └── test-postgres-database.yaml      # Example manifests
├── postgres-self-service-platform.md    # Design documentation
└── README.md                           # This file
```

## Related Repositories

- **[postgres-database-controller](https://github.com/AgnosticDBA/postgres-database-controller)** - Go operator that manages PostgresDatabase resources
- **[percona-postgresql-operator](https://github.com/percona/percona-postgresql-operator)** - Underlying PostgreSQL operator

## License

Apache License 2.0