# OpenShift & ACM Day 2 Operations Guide
*Comprehensive Step-by-Step Configuration Guide*

---

## Table of Contents
1. [Health Check](#1-health-check)
2. [Certificate Configuration](#2-certificate-configuration)
3. [Identity & Access Management](#3-identity--access-management)
4. [Storage Configuration (Pure Storage & NetApp S3)](#4-storage-configuration-pure-storage--netapp-s3)
5. [Backup (OADP)](#5-backup-oadp)
6. [Node and Capacity Scaling](#6-node-and-capacity-scaling)
7. [Platform & User Workload Monitoring and Logging](#7-platform-and-user-workload-monitoring-and-logging)
8. [Security Compliance, Vulnerability, and File Auditing](#8-security-compliance-vulnerability-and-file-auditing)
9. [Multi-Cluster Observability Aggregation](#9-multi-cluster-observability-aggregation)
10. [Global Application Lifecycle Management](#10-global-application-lifecycle-management)
11. [Multi-Cluster Networking & Interconnect](#11-multi-cluster-networking--interconnect)

---

## 1. Health Check

### Cluster Health
Before proceeding with Day 2 operations, ensure the cluster is entirely healthy. All Cluster Operators must be `Available=True`, `Progressing=False`, and `Degraded=False`.

```bash
# Check all cluster operators
oc get clusteroperators

# Check all nodes
oc get nodes -o wide

# Check for any degraded components
oc get clusteroperators | grep -v 'True.*False.*False'
```

### Etcd Backup
Perform a manual backup of the etcd database. It is recommended to schedule this via cron on a bastion host or automated pipeline.

```bash
# Start a debug pod on a master node
oc debug node/<master-node-name>

# Once inside the shell, change root
chroot /host

# Run the cluster-backup script
/usr/local/bin/cluster-backup.sh /home/core/assets/backup

# Copy the backup files off the node to your local machine
exit
exit
oc cp <debug-pod-name>:/host/home/core/assets/backup ./etcd-backup
```

### Create Must-Gather
Generate a must-gather bundle for proactive support or troubleshooting.

```bash
oc adm must-gather --dest-dir=./must-gather-default
```

## 2. Certificate Configuration

### Install API Certificate
Replace the default API server certificate with a trusted CA-signed certificate.

```bash
# 1. Create a secret with your TLS cert and key in the openshift-config namespace
oc create secret tls api-tls-cert \
  --cert=</path/to/cert.crt> \
  --key=</path/to/cert.key> \
  -n openshift-config

# 2. Patch the apiserver cluster configuration
oc patch apiserver cluster \
  --type=merge -p \
  '{"spec":{"servingCerts": {"namedCertificates": [{"names": ["api.<cluster-name>.<base-domain>"], "servingCertificate": {"name": "api-tls-cert"}}]}}}'
```

### Install Ingress Certificate

```bash
# 1. Create a TLS secret in the openshift-ingress namespace
oc create secret tls ingress-tls-cert \
  --cert=</path/to/wildcard.crt> \
  --key=</path/to/wildcard.key> \
  -n openshift-ingress

# 2. Patch the default ingresscontroller
oc patch ingresscontroller default \
  -n openshift-ingress-operator \
  --type=merge \
  -p '{"spec":{"defaultCertificate": {"name": "ingress-tls-cert"}}}'
```

### IDP / LDAP Configuration
Integrate OpenShift with Active Directory/LDAP.

```yaml
oc create secret generic ldap-bind-password \
  --from-literal=bindPassword='<SVC_OVE_PASSWORD>' \
  -n openshift-config
```

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: sheetz_ldap
    mappingMethod: claim
    type: LDAP
    ldap:
      attributes:
        id:
        - dn
        email:
        - mail
        name:
        - cn
        preferredUsername:
        - sAMAccountName
      bindDN: "CN=svc_ove,OU=ServiceAccounts,DC=sheetzad,DC=sheetz,DC=com" 
      bindPassword:
        name: ldap-bind-password
      insecure: false
      url: "ldaps://sheetzad.sheetz.com:636/DC=sheetzad,DC=sheetz,DC=com?sAMAccountName"
```

### Groups / User Sync

```yaml
kind: LDAPSyncConfig
apiVersion: v1
url: "ldaps://sheetzad.sheetz.com:636"
insecure: false
bindDN: "CN=svc_ove,OU=ServiceAccounts,DC=sheetzad,DC=sheetz,DC=com"
bindPassword:
  file: "/etc/secrets/bindPassword"
activeDirectory:
  usersQuery:
    baseDN: "DC=sheetzad,DC=sheetz,DC=com"
    scope: sub
    derefAliases: never
    filter: "(objectClass=user)"
  userNameAttributes: [ sAMAccountName ]
  groupMembershipAttributes: [ member ]
groupUIDNameMapping:
  "CN=OVEClusterAdmins,OU=OVE,OU=SheetzHighlyPrivilegedGroups,DC=sheetzad,DC=sheetz,DC=com": OVEClusterAdmins
  "CN=OVEClusterViewers,OU=OVE,OU=SheetzHighlyPrivilegedGroups,DC=sheetzad,DC=sheetz,DC=com": OVEClusterViewers
```

```bash
# Create ldap-sync.yaml and execute the sync
oc adm groups sync --sync-config=ldap-sync.yaml --confirm
```

### cluster-admin RBAC Permissions

```bash
# Assign cluster-admin rights to OVEClusterAdmins 
oc adm policy add-cluster-role-to-group cluster-admin OVEClusterAdmins

# Assign view/read-only rights to OVEClusterViewers
oc adm policy add-cluster-role-to-group cluster-reader OVEClusterViewers
```

## 4. Storage Configuration

### Pure Storage CSI Configuration (Block and File)
Install the Pure Storage Operator from OperatorHub.
Create the connection secret for FlashArray (Block) and FlashBlade (File).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pure-provisioner-secret
  namespace: pure-csi
type: Opaque
stringData:
  pure.json: |
    {
      "FlashArrays": [
        {
          "MgmtEndPoint": "<IP_O_FQDN_GESTION_FLASHARRAY>",
          "APIToken": "<API_TOKEN_FLASHARRAY>"
        }
      ],
      "FlashBlades": [
        {
          "MgmtEndPoint": "<IP_O_FQDN_GESTION_FLASHBLADE>",
          "APIToken": "<API_TOKEN_FLASHBLADE>",
          "NfsEndPoint": "<IP_DATOS_NFS_FLASHBLADE>"
        }
      ]
    }
```

Example

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pure-provisioner-secret
  namespace: pure-csi
type: Opaque
stringData:
  pure.json: |
    {
      "FlashArrays": [{"MgmtEndPoint": "10.0.0.1", "APIToken": "fa-api-token"}],
      "FlashBlades": [{"MgmtEndPoint": "10.0.0.2", "APIToken": "fb-api-token", "NfsEndPoint": "10.0.0.3"}]
    }
```

### Pure StorageClass for Block (ReadWriteOnce)
Used for databases and high-performance single-node workloads.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: <NOMBRE_STORAGECLASS_BLOCK>
provisioner: pure-csi
parameters:
  backend: "block"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### Pure StorageClass for File System (ReadWriteMany)
Used for shared volumes across multiple pods (NFS on FlashBlade).

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: <NOMBRE_STORAGECLASS_FILE>
provisioner: pure-csi
parameters:
  backend: "file"
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

### Pure Storage FlashBlade S3 Integration (Step-by-Step)
**Architectural Note:** Pure Storage FlashBlade natively delivers high-performance, S3-compatible Object Storage. Object Storage (S3) does not use standard CSI drivers (Block/File PV/PVCs). Instead, OpenShift applications and platform operators consume FlashBlade S3 via direct S3 API endpoints and service credentials.

#### Step 1: Provision S3 Bucket & Credentials on Pure FlashBlade
- Log in to the Pure Storage FlashBlade Management UI or CLI.
- Create a dedicated Object Store Account (e.g., openshift-s3-account).
- Generate an Access Key and Secret Key for the S3 service user.
- Create the required S3 buckets (e.g., oadp-backups, loki-logs, thanos-metrics).

#### Step 2: Create OpenShift Credentials Secrets
Generate the Kubernetes Secret inside the target namespaces (e.g., openshift-adp for backups, openshift-logging for Loki).

```bash
# 1. Prepare credential file format
cat << EOF > pure-s3-credentials
[default]
aws_access_key_id=PURE_FLASHBLADE_S3_ACCESS_KEY
aws_secret_access_key=PURE_FLASHBLADE_S3_SECRET_KEY
EOF

# 2. Create OpenShift Secret for OADP (openshift-adp namespace)
oc create secret generic pure-s3-creds \
  -n openshift-adp \
  --from-file cloud=pure-s3-credentials

# 3. Create OpenShift Secret for Loki Logging (openshift-logging namespace)
oc create secret generic pure-s3-loki-secret \
  -n openshift-logging \
  --from-literal=access_key_id=PURE_FLASHBLADE_S3_ACCESS_KEY \
  --from-literal=access_key_secret=PURE_FLASHBLADE_S3_SECRET_KEY \
  --from-literal=endpoint=http://<FLASHBLADE_S3_DATA_VIP>:8080 \
  --from-literal=bucketnames=loki-logs
```

#### Step 3: Configure Applications/Operators to Endpoint
Point your OpenShift components to the FlashBlade S3 Data VIP/DNS endpoint (e.g., http://10.0.0.2:8080 or https://s3.pureflashblade.example.com).

### Using S3 from a NetApp Array (ONTAP S3 / StorageGRID)
**CSI Architecture Note:** Object Storage (S3) from NetApp is also consumed directly via S3 API endpoints rather than standard CSI PV/PVC mounts.

**Step-by-Step NetApp S3 usage for OpenShift:**
- Enable ONTAP S3 on your NetApp SVM or configure StorageGRID.
- Generate S3 Access Key and Secret Key on the NetApp system.
- Create an OpenShift Secret using the NetApp S3 credentials:

```bash
cat << EOF > netapp-s3-credentials
[default]
aws_access_key_id=NETAPP_S3_ACCESS_KEY
aws_secret_access_key=NETAPP_S3_SECRET_KEY
EOF

# Create secret for components like OADP (Backup) or Thanos/Loki
oc create secret generic netapp-s3-creds -n openshift-adp --from-file cloud=netapp-s3-credentials
```

## 5. Backup (OADP)
Configure OpenShift API for Data Protection (OADP) using either Pure FlashBlade S3 or NetApp ONTAP S3 as the backup location.

```yaml
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: oadp-pure-or-netapp-s3
  namespace: openshift-adp
spec:
  configuration:
    velero:
      defaultPlugins:
        - openshift
        - aws
        - csi
      featureFlags:
        - EnableCSI
  backupLocations:
    - velero:
        config:
          profile: "default"
          region: "us-east-1"
          # Replace with your Pure FlashBlade or NetApp S3 endpoint URL
          s3Url: "http://<FLASHBLADE_OR_NETAPP_S3_ENDPOINT>:8080"
          insecureSkipTLSVerify: "true"
          s3ForcePathStyle: "true"
        credential:
          key: cloud
          name: pure-s3-creds # or netapp-s3-creds
        objectStorage:
          bucket: oadp-backups
          prefix: velero
        provider: aws
        default: true
```

## 6. Node and Capacity Scaling

### Node Labeling

```bash
oc label node <node-name> node-role.kubernetes.io/infra=
oc adm taint nodes <node-name> node-role.kubernetes.io/infra=reserved:NoSchedule
```

### Baremetal Operator

```yaml
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: worker-0
  namespace: openshift-machine-api
spec:
  bmc:
    address: ipmi://10.0.0.100
    credentialsName: worker-0-bmc-secret
  bootMACAddress: 00:11:22:33:44:55
  online: true
```

## 7. Platform and User Workload Monitoring and Logging

### Monitoring ConfigMap
Enable User Workload monitoring and use Pure Block for Thanos/Prometheus PVCs.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
    prometheusK8s:
      retention: 15d
      volumeClaimTemplate:
        spec:
          storageClassName: pure-block
          resources:
            requests:
              storage: 100Gi
    alertmanagerMain:
      volumeClaimTemplate:
        spec:
          storageClassName: pure-block
          resources:
            requests:
              storage: 20Gi
```

### Configured Alertmanager

```yaml
# Inside the alertmanager.yaml
global:
  resolve_timeout: 5m
route:
  receiver: 'slack-notifications'
receivers:
- name: 'slack-notifications'
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
    channel: '#openshift-alerts'
```

### OpenShift Logging & Loki Operator
Loki requires Object Storage. This is configured using Pure FlashBlade S3 or NetApp S3.

```yaml
apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
  name: logging-loki
  namespace: openshift-logging
spec:
  size: 1x.small
  storage:
    schemas:
    - version: v12
      effectiveDate: "2023-01-01"
    secret:
      name: pure-s3-loki-secret # Pure FlashBlade S3 or NetApp S3 Secret
      type: s3
  storageClassName: pure-block
```

### ClusterLogForwarder

```yaml
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: openshift-logging
spec:
  outputs:
   - name: default-lokistack
     type: loki
     url: https://logging-loki-gateway-http.openshift-logging.svc:8080
  pipelines:
   - name: all-logs
     inputRefs:
     - infrastructure
     - application
     outputRefs:
     - default-lokistack
```

## 8. Security Compliance, Vulnerability, and File Auditing

### Compliance Operator

```yaml
apiVersion: compliance.openshift.io/v1alpha1
kind: ScanSettingBinding
metadata:
  name: cis-compliance
  namespace: openshift-compliance
profiles:
  - name: ocp4-cis-node
    kind: Profile
    apiGroup: compliance.openshift.io/v1alpha1
settingsRef:
  name: default
  kind: ScanSetting
  apiGroup: compliance.openshift.io/v1alpha1
```

### File Integrity Operator

```yaml
apiVersion: fileintegrity.openshift.io/v1alpha1
kind: FileIntegrity
metadata:
  name: worker-fileintegrity
  namespace: openshift-file-integrity
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  config:
    name: "m-worker-aides"
    namespace: "openshift-file-integrity"
    key: "config"
```

## 9. Multi-Cluster Observability Aggregation
Configure Thanos in ACM to use your Pure FlashBlade S3 or NetApp S3 backend for global metrics retention.

```yaml
apiVersion: observability.open-cluster-management.io/v1beta2
kind: MultiClusterObservability
metadata:
  name: observability
spec:
  observabilityAddonSpec:
    enableMetrics: true
    interval: 60
  storageConfig:
    metricObjectStorage:
      name: thanos-pure-s3-secret
      key: thanos.yaml
```

## 10. Global Application Lifecycle Management

### OpenShift GitOps Operator
Install OpenShift GitOps from OperatorHub.

### Create GitOps Repo via Application CR

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-global-app
  namespace: openshift-gitops
spec:
  destination:
    namespace: my-app-prod
    server: https://kubernetes.default.svc
  project: default
  source:
    path: kustomize/overlays/prod
    repoURL: https://github.com/my-org/my-gitops-repo.git
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## 11. Multi-Cluster Networking & Interconnect

### Submariner Add-on
In ACM, Submariner connects overlay networks.

#### Open UDP Ports on Gateway Nodes
Ensure your physical firewalls allow:
- UDP 4500 and UDP 500 (IPsec)
- UDP 51820 (WireGuard)

Label the dedicated gateway nodes:

```bash
oc label node <gateway-node> submariner.io/gateway=true
```

#### Configured Placement and PlacementRule CRDs

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: submariner-placement
  namespace: default
spec:
  clusterSets:
    - prod-clusters
---
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: submariner
  namespace: spoke-cluster-1
spec:
  installNamespace: submariner-operator
```
