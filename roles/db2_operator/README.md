# Ansible Role: db2_operator

Deploy IBM Db2 instances using the Db2 Operator on OpenShift/Kubernetes.

## Description

This role automates the deployment of IBM Db2 instances using the Db2 Operator with Db2uInstance CR, including:
- Installation of the Db2 Operator (requires IBM Catalog to be pre-installed)
- Creation of Db2 instance using Db2uInstance custom resource
- Configuration of storage, resources, and security
- Namespace and secret management
- Database initialization

**Important:** This role requires the IBM Operator Catalog to be installed first. Use the `ibm_catalog` role before running this role.

**Note:** This role uses the Db2uInstance custom resource (CR) instead of the deprecated Db2uCluster CR for better compatibility with newer Db2 operator versions.

## Requirements

- OpenShift 4.12+ or Kubernetes 1.27+
- IBM Entitlement Registry Key
- IBM Operator Catalog installed (use `ibm_catalog` role)
- Ansible 2.9+
- kubernetes.core collection
- Sufficient cluster resources (minimum 8Gi memory, 2 CPU cores)

## Role Variables

### Required Variables

```yaml
entitled_registry_key: ""           # IBM Entitlement Registry Key
db2_namespace: "db2u"               # Namespace for Db2 instance (default: db2u)
db2_instance_name: "db2inst1"       # Db2 instance name
```

### Optional Variables

#### Operator Configuration
```yaml
db2_operator_namespace: "ibm-common-services"
db2_operator_channel: "v110509.0"
db2_operator_source: "ibm-operator-catalog"
```

#### Instance Configuration
```yaml
db2_version: "11.5.9.0"
db2_workload: "ANALYTICS"           # ANALYTICS, OLTP, or SAP
```

#### Database Configuration
```yaml
db2_database_name: "BLUDB"
db2_database_territory: "US"
db2_database_pagesize: "32768"
```

#### Storage Configuration
```yaml
# Intelligent storage class selection
storage_class_rwo: ""               # ReadWriteOnce - for data, logs, temp (auto-detect if empty)
storage_class_rwx: ""               # ReadWriteMany - for meta, backup, archive (auto-detect if empty)
db2_storage_class: ""               # Override both RWO and RWX if specified

# Storage sizes
db2_meta_storage_size: "20Gi"
db2_data_storage_size: "100Gi"
db2_backup_storage_size: "100Gi"
db2_logs_storage_size: "100Gi"
db2_temp_storage_size: "100Gi"
```

**Storage Class Auto-Detection:**
The role automatically detects appropriate storage classes based on your cluster provider:
- IBM Cloud: `ibmc-block-gold` (RWO), `ibmc-file-gold-gid` (RWX)
- Red Hat OpenShift: `ocs-storagecluster-ceph-rbd` (RWO), `ocs-storagecluster-cephfs` (RWX)
- AWS: `gp2` (RWO), `efs` (RWX)
- Azure: `managed-premium` (RWO), `azurefile-premium` (RWX)
- TechZone: `managed-nfs-storage` (both)
- IBM Spectrum Scale: `scale-cnsa` (both)

You can override auto-detection by setting `STORAGE_CLASS_RWO` and `STORAGE_CLASS_RWX` environment variables.

#### Resource Configuration
```yaml
db2_cpu_requests: "2000m"
db2_cpu_limits: "4000m"
db2_memory_requests: "8Gi"
db2_memory_limits: "16Gi"
```

#### Credentials
```yaml
db2_admin_password: "passw0rd"
db2_instance_password: "passw0rd"
```

#### License
```yaml
db2_license_accept: true
db2_license_model: "BYOL"           # BYOL or Virtual Processor Core
```

#### Advanced Configuration
```yaml
db2_ssl_enabled: true
db2_hadr_enabled: false
db2_logarchmeth1: "DISK:/mnt/logs/archive"
db2_logfilsiz: "16384"
db2_logsecond: "12"
```

## Dependencies

- `ibm_catalog` role (must be run first to install IBM Operator Catalog)

## Example Playbook

### Complete Deployment with IBM Catalog

```yaml
---
- name: Deploy IBM Db2 with Catalog
  hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - ibm_catalog      # Install IBM Operator Catalog first
    - db2_operator     # Deploy Db2 instance
```

### Deploy Db2 Instance Only (Catalog Already Installed)

```yaml
---
- name: Deploy IBM Db2 Instance
  hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - db2_operator
```

### Custom Configuration

```yaml
---
- name: Deploy IBM Db2 with Custom Settings
  hosts: localhost
  connection: local
  gather_facts: false
  vars:
    db2_namespace: "db2u"
    db2_instance_name: "mydb2"
    db2_version: "11.5.9.0"
    db2_workload: "ANALYTICS"
    db2_data_storage_size: "200Gi"
    db2_cpu_limits: "8000m"
    db2_memory_limits: "32Gi"
  roles:
    - ibm_catalog
    - db2_operator
```

## Usage

### Basic Instance Deployment

```bash
# Set environment variables
export ENTITLED_REGISTRY_KEY="your-entitlement-key"
export DB2_NAMESPACE="db2u"
export DB2_INSTANCE_NAME="db2inst1"

# Run playbook
ansible-playbook playbooks/setup-db2.yml
```

### Custom Instance Configuration

```bash
export ENTITLED_REGISTRY_KEY="your-key"
export DB2_NAMESPACE="db2u"
export DB2_INSTANCE_NAME="mydb2"
export DB2_VERSION="11.5.9.0"
export DB2_WORKLOAD="ANALYTICS"
export DB2_DATA_STORAGE_SIZE="200Gi"
export DB2_CPU_LIMITS="8000m"
export DB2_MEMORY_LIMITS="32Gi"

ansible-playbook playbooks/setup-db2.yml
### Manual Storage Class Configuration

```bash
export ENTITLED_REGISTRY_KEY="your-key"
export DB2_NAMESPACE="db2u"
export DB2_INSTANCE_NAME="db2inst1"
# Manually specify storage classes
export STORAGE_CLASS_RWO="my-block-storage"
export STORAGE_CLASS_RWX="my-file-storage"

ansible-playbook playbooks/setup-db2.yml
```
```

## Post-Deployment

### Connect to Db2 Instance

```bash
# Get the pod name
kubectl get pods -n db2u

# Connect to Db2 instance
kubectl exec -it <pod-name> -n db2u -- su - db2inst1
```

### Get Instance Password

```bash
kubectl get secret db2-instance-secret -n db2u -o jsonpath='{.data.password}' | base64 -d
```

### Connect to Database

```bash
kubectl exec -it <pod-name> -n db2u -- su - db2inst1 -c 'db2 connect to BLUDB'
```

### Instance Information

The role creates an instance info file at `/tmp/db2_instance_info_<instance-name>.txt` with:
- Instance details
- Connection information
- Credentials location
- Useful commands

## Verification

### Check Db2 Status

```bash
# Check Db2uInstance status
kubectl get db2uinstance -n db2u

# Check pods
kubectl get pods -n db2u

# Check services
kubectl get svc -n db2u

# Check PVCs
kubectl get pvc -n db2u
```

### Check Operator Status

```bash
# Check subscription
kubectl get subscription -n ibm-common-services

# Check CSV
kubectl get csv -n ibm-common-services | grep db2u
```

## Troubleshooting

### Catalog Not Found

If you get an error about the IBM Operator Catalog not being found:

```bash
# Install the catalog first
ansible-playbook playbooks/tools/deploy_ibm_catalog.yml
```

### Operator Not Installing

```bash
# Check catalog source
kubectl get catalogsource -n openshift-marketplace

# Check subscription
kubectl get subscription -n ibm-common-services

# Check CSV
kubectl get csv -n ibm-common-services
```

### Instance Not Starting

```bash
# Check Db2uInstance status
kubectl describe db2uinstance <instance-name> -n db2u

# Check pod logs
kubectl logs <pod-name> -n db2u

# Check events
kubectl get events -n db2u --sort-by='.lastTimestamp'
```

### Storage Issues

```bash
# Check PVCs
kubectl get pvc -n db2u

# Check storage class
kubectl get storageclass

# Describe PVC
kubectl describe pvc <pvc-name> -n db2u
```

### Pod Not Ready

```bash
# Check pod status
kubectl get pods -n db2u

# Describe pod
kubectl describe pod <pod-name> -n db2u

# Check logs
kubectl logs <pod-name> -n db2u -c db2u
```

## Features

- **Automated Deployment**: Complete Db2 instance deployment with minimal configuration
- **Flexible Storage**: Auto-detection of storage classes or custom configuration
- **Resource Management**: Configurable CPU and memory limits
- **Security**: Automatic secret management for credentials
- **Database Initialization**: Creates database with specified configuration
- **Workload Optimization**: Support for ANALYTICS, OLTP, and SAP workloads
- **SSL Support**: Optional SSL/TLS encryption
- **Comprehensive Logging**: Detailed deployment information and connection details

## Default Namespace

This role uses `db2u` as the default namespace for Db2 instances, following IBM's recommended naming convention for Db2 Universal deployments.

## License

Apache-2.0

## Author

IBM Sterling DevOps Team

## Made with Bob