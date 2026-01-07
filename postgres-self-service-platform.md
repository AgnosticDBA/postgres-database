# PostgreSQL Self-Service Platform: Developer-Friendly CRD Design

## The Problem (Reminder)

Your working `cr3.yaml` is 50+ lines of YAML that developers need to understand:

```yaml
apiVersion: pgv2.percona.com/v2
kind: PerconaPGCluster
metadata:
  name: postgres-test
  namespace: percona-postgresql-operator
spec:
  crVersion: 2.8.2
  image: docker.io/percona/percona-distribution-postgresql:17.7-2
  imagePullPolicy: Always
  postgresVersion: 17
  instances:
  - name: instance1
    replicas: 3
    affinity:
      podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          podAffinityTerm:
            labelSelector:
              matchLabels:
                postgres-operator.crunchydata.com/data: postgres
            topologyKey: kubernetes.io/hostname
    dataVolumeClaimSpec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: standard
  proxy:
    pgBouncer:
      replicas: 3
      image: docker.io/percona/percona-pgbouncer:1.25.0-1
      affinity:
        # ... more complexity
  backups:
    pgbackrest:
      image: docker.io/percona/percona-pgbackrest:2.57.0-1
      repoHost:
        affinity:
          # ... more complexity
      # ... more configuration
  pmm:
    enabled: false
    image: docker.io/percona/pmm-client:3.5.0
    secret: cluster1-pmm-secret
    serverHost: monitoring-service
```

**Developers shouldn't need to understand:**
- `crVersion`
- Image URIs and versions
- Affinity rules
- PgBouncer configuration
- pgBackRest configuration
- PMM setup
- Storage class names
- Kubernetes topology rules

---

## The Solution: PostgresDatabase CRD

Create a **simple developer-facing CRD** that abstracts all complexity:

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

**That's it!** Developers write 8 lines. Your platform handles the rest.

---

## Three-Layer Architecture

### Layer 1: Developer Experience

```yaml
apiVersion: databases.mycompany.com/v1
kind: PostgresDatabase
metadata:
  name: my-database
  namespace: my-namespace
spec:
  version: 17              # PostgreSQL version
  replicas: 3              # 1 = single, 3 = HA with failover
  storage: 100Gi           # Storage size
  backup: true             # Enable automated backups
  monitoring: true         # Enable PMM monitoring
```

**What developers see. Simple, clear, intentional.**

### Layer 2: Platform Controller

Your custom controller watches `PostgresDatabase` resources and translates them:

```go
// Pseudo-code showing the concept
func (r *PostgresDatabaseReconciler) Reconcile(db *PostgresDatabase) {
  // Generate full PerconaPGCluster from simple PostgresDatabase
  cluster := &PerconaPGCluster{
    Spec: PerconaPGClusterSpec{
      CRVersion: "2.8.2",
      Image: fmt.Sprintf("docker.io/percona/percona-distribution-postgresql:%d.7-2", db.Spec.Version),
      PostsVersion: db.Spec.Version,
      
      Instances: []Instance{{
        Name: "instance1",
        Replicas: db.Spec.Replicas,
        DataVolumeClaimSpec: VolumeSpec{
          Storage: db.Spec.Storage,
          StorageClassName: "gp2",  // Platform default
        },
        Affinity: podAntiAffinity(),  // Auto-configured for HA
      }},
      
      Proxy: PgBouncer{
        Replicas: db.Spec.Replicas,
        Image: "docker.io/percona/percona-pgbouncer:1.25.0-1",
        Affinity: podAntiAffinity(),
      },
      
      Backups: Backups{
        Enabled: db.Spec.Backup,
        Image: "docker.io/percona/percona-pgbackrest:2.57.0-1",
        Repos: []Repo{{Name: "repo1"}},
      },
      
      PMM: PMM{
        Enabled: db.Spec.Monitoring,
        Image: "docker.io/percona/pmm-client:3.5.0",
        ServerHost: "prometheus.monitoring",
      },
    },
  }
  
  k8s.Create(cluster)
}
```

### Layer 3: Infrastructure

```yaml
apiVersion: pgv2.percona.com/v2
kind: PerconaPGCluster
metadata:
  name: my-database
  namespace: my-namespace
  labels:
    created-by: "postgres-database-controller"
spec:
  crVersion: 2.8.2
  image: docker.io/percona/percona-distribution-postgresql:17.7-2
  postgresVersion: 17
  instances:
  - name: instance1
    replicas: 3
    affinity:
      podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          podAffinityTerm:
            labelSelector:
              matchLabels:
                postgres-operator.crunchydata.com/data: postgres
            topologyKey: kubernetes.io/hostname
    dataVolumeClaimSpec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 100Gi
      storageClassName: gp2
  proxy:
    pgBouncer:
      replicas: 3
      image: docker.io/percona/percona-pgbouncer:1.25.0-1
      affinity:
        # ... auto-configured
  backups:
    pgbackrest:
      image: docker.io/percona/percona-pgbackrest:2.57.0-1
      # ... auto-configured
  pmm:
    enabled: true
    image: docker.io/percona/pmm-client:3.5.0
    serverHost: prometheus.monitoring
```

**What actually runs. Full complexity managed by the platform.**

---

## Implementation Phases

### Phase 1: CRD Definition (2 hours)

Create the CRD YAML:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresdatabases.databases.mycompany.com
spec:
  group: databases.mycompany.com
  names:
    kind: PostgresDatabase
    plural: postgresdatabases
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required:
            - version
            - replicas
            - storage
            properties:
              version:
                type: integer
                minimum: 13
                maximum: 17
                description: "PostgreSQL version (13, 14, 15, 16, 17)"
              replicas:
                type: integer
                minimum: 1
                maximum: 10
                description: "Number of replicas (1 = single, 3+ = HA)"
              storage:
                type: string
                pattern: '^\d+Gi$'
                description: "Storage size (e.g., 100Gi)"
              backup:
                type: boolean
                default: true
                description: "Enable automated backups"
              monitoring:
                type: boolean
                default: true
                description: "Enable PMM monitoring"
```

### Phase 2: Controller Implementation (1 week)

Build a Kubernetes operator that:
- Watches `PostgresDatabase` resources
- Generates `PerconaPGCluster` automatically
- Applies best practices (HA, backups, monitoring)
- Handles updates and deletions

**Technology options:**
- **Go + Operator SDK** (recommended) - 3-4 days
- **Rust + Kopium** - 3-4 days
- **Python + kopf** - 2-3 days

### Phase 3: Kyverno Policies (4 hours)

Enforce compliance even if developers try to bypass:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: postgres-database-required
spec:
  validationFailureAction: audit  # or enforce
  rules:
  - name: require-postgres-database
    match:
      resources:
        kinds:
        - PerconaPGCluster
    validate:
      message: "Use PostgresDatabase CRD instead of PerconaPGCluster directly"
      pattern:
        metadata:
          labels:
            created-by: "postgres-database-controller"
```

### Phase 4: Documentation & Rollout (2 hours)

Developer documentation:

```markdown
# Creating a PostgreSQL Database

1. Create a manifest:

```yaml
apiVersion: databases.mycompany.com/v1
kind: PostgresDatabase
metadata:
  name: my-app-db
  namespace: production
spec:
  version: 17
  replicas: 3
  storage: 100Gi
  backup: true
  monitoring: true
```

2. Apply it:

```bash
kubectl apply -f postgres-db.yaml
```

3. Wait for it to be ready:

```bash
kubectl get postgres my-app-db -w
```

4. Connect:

```bash
kubectl port-forward svc/my-app-db 5432:5432
psql -h localhost -U postgres
```

Done! The platform handles:
- ✅ Image versions
- ✅ HA configuration
- ✅ Automated backups to S3
- ✅ PMM monitoring
- ✅ Resource limits
- ✅ Pod anti-affinity
```

---

## Timeline & Effort

| Phase | Time | Effort | Owner |
|-------|------|--------|-------|
| CRD Design | 2h | Easy | You |
| Controller | 3-4d | Medium | Contractor or You |
| Policies | 4h | Easy | You |
| Documentation | 2h | Easy | You |
| **Total** | **~1 week** | **Medium** | - |

---

## Benefits After Implementation

| Scenario | Before | After |
|----------|--------|-------|
| New developer creates database | Read 50-line YAML, configure all fields, make mistakes | Copy template, change name, done ✅ |
| Update PostgreSQL version | Manually edit `spec.image`, update all pods | Update controller, automatic rollout ✅ |
| Add monitoring to existing cluster | Manual PMM setup for each cluster | Already enabled by default ✅ |
| Enforce compliance | Manual code reviews | Kyverno blocks non-compliant resources ✅ |
| Change storage class | Edit each cluster's YAML | Update controller default once ✅ |

---

## Immediate Next Steps

1. **Today:** Verify PostgreSQL cluster works with corrected setup
2. **Tomorrow:** Design the PostgresDatabase CRD (30 min)
3. **Week 4:** Build the controller (3-4 days)
4. **Week 4:** Deploy Kyverno policies (4 hours)
5. **Week 5:** GitOps integration

---

## PostgresDatabase CRD Specification (Draft)

Save this for development:

```yaml
# crd-postgres-database.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresdatabases.databases.mycompany.com
  annotations:
    description: "Self-service PostgreSQL databases"
spec:
  group: databases.mycompany.com
  names:
    kind: PostgresDatabase
    plural: postgresdatabases
    shortNames:
    - pgdb
    - postgres
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          apiVersion:
            type: string
          kind:
            type: string
          metadata:
            type: object
          spec:
            type: object
            required:
            - version
            - replicas
            - storage
            properties:
              version:
                type: integer
                minimum: 13
                maximum: 17
                description: "PostgreSQL major version"
                enum: [13, 14, 15, 16, 17]
              replicas:
                type: integer
                minimum: 1
                maximum: 10
                default: 1
                description: "Number of database replicas (1 = single, 3 = HA)"
              storage:
                type: string
                pattern: '^\d+(Gi|Ti)$'
                description: "Storage capacity (e.g., 100Gi, 1Ti)"
              backup:
                type: boolean
                default: true
                description: "Enable automated backups to S3"
              monitoring:
                type: boolean
                default: true
                description: "Enable PMM monitoring"
          status:
            type: object
            properties:
              phase:
                type: string
                enum: [Pending, Creating, Ready, Failed]
              endpoint:
                type: string
              replicas:
                type: integer
              message:
                type: string
    subresources:
      status: {}
    additionalPrinterColumns:
    - name: Version
      type: integer
      jsonPath: .spec.version
    - name: Replicas
      type: integer
      jsonPath: .spec.replicas
    - name: Storage
      type: string
      jsonPath: .spec.storage
    - name: Status
      type: string
      jsonPath: .status.phase
    - name: Endpoint
      type: string
      jsonPath: .status.endpoint
```

---

## Ready to Build?

You have:
- ✅ Working PostgreSQL on minikube
- ✅ Understanding of the complexity to abstract
- ✅ Clear architecture (3 layers)
- ✅ CRD specification
- ✅ Timeline (1 week)

Next decision: **Do you want to build the controller yourself, or should we outline it for a contractor?**

Either way, this abstraction layer will transform your platform from "working" to "production-ready for developers." 🚀