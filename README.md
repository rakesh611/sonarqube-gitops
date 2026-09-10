# SonarQube on OpenShift — GitOps Deployment & Complete Troubleshooting Runbook

## 1. Purpose

This document records the complete deployment of **SonarQube + PostgreSQL on OpenShift using Argo CD and Kustomize**.

It is written as a reusable L2/L3 operational runbook so that the complete deployment and troubleshooting process can be repeated from scratch.

The deployment covers:

* OpenShift namespace
* Argo CD Application
* Kustomize
* RBAC for Argo CD
* PostgreSQL
* PostgreSQL persistent storage
* SonarQube
* SonarQube persistent storage
* Service
* OpenShift Route
* DNS
* TLS
* Application verification
* Common failure scenarios
* RCA and fixes

---

# 2. Environment

## OpenShift

```text
OpenShift: 4.20.x
Kubernetes: v1.33.6
```

## Nodes

```text
Control Plane:
192.168.22.201
192.168.22.202
192.168.22.203

Workers:
192.168.22.211
192.168.22.212
192.168.22.213
```

## Storage

```text
StorageClass: lvms-vg1
Provisioner: topolvm.io
Access Mode: RWO
```

## Namespaces

```text
Application:
sonarqube

Argo CD:
openshift-gitops
```

## DNS

Lab DNS/BIND:

```text
192.168.22.1
```

---

# 3. Final Architecture

```text
                         GitHub
                           |
                           |
                           v
                 +-------------------+
                 |      Argo CD      |
                 | openshift-gitops  |
                 +---------+---------+
                           |
                           | Kustomize
                           v
              +-------------------------+
              |     OpenShift Cluster   |
              |                         |
              | Namespace: sonarqube    |
              |                         |
              |   +----------------+    |
              |   |   SonarQube    |    |
              |   |     :9000      |    |
              |   +-------+--------+    |
              |           | JDBC         |
              |           v              |
              |   +----------------+    |
              |   |   PostgreSQL   |    |
              |   |     :5432      |    |
              |   +----------------+    |
              |           |              |
              |          PVC             |
              |           |              |
              |       lvms-vg1           |
              +-----------+--------------+
                          |
                          v
                  OpenShift Router
                          |
                          v
          sonarqube-sonarqube.apps.lab.ocp.lan
```

---

# 4. Git Repository Structure

Repository:

```text
https://github.com/rakesh611/sonarqube-gitops.git
```

Structure:

```text
sonarqube-gitops/
│
├── argocd/
│   └── sonarqube-application.yaml
│
├── namespace/
│   └── namespace.yaml
│
├── postgres/
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   ├── pvc.yaml
│   ├── secret.yaml
│   └── service.yaml
│
├── sonarqube/
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   ├── pvc.yaml
│   ├── route.yaml
│   ├── secret.yaml
│   └── service.yaml
│
├── kustomization.yaml
└── README.md
```

---

# 5. Root Kustomization

Root:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - namespace/namespace.yaml
  - postgres/
  - sonarqube/
```

This tells Kustomize to deploy:

```text
Namespace
PostgreSQL resources
SonarQube resources
```

---

# 6. SonarQube Kustomization

`sonarqube/kustomization.yaml` must include the Route.

Example:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - configmap.yaml
  - secret.yaml
  - pvc.yaml
  - deployment.yaml
  - service.yaml
  - route.yaml
```

Important:

If `route.yaml` exists in Git but is not listed here, Kustomize will not render it.

---

# 7. PostgreSQL Kustomization

Example:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - secret.yaml
  - pvc.yaml
  - deployment.yaml
  - service.yaml
```

---

# 8. Argo CD Application

The Argo CD Application runs in:

```text
openshift-gitops
```

Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: sonarqube
  namespace: openshift-gitops

spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: sonarqube

  project: default

  source:
    repoURL: https://github.com/rakesh611/sonarqube-gitops.git
    path: .
    targetRevision: HEAD

  syncPolicy:
    automated:
      enabled: true
      prune: true
      selfHeal: true
```

---

# 9. Deploy Application

Login to OpenShift:

```bash
oc login <api-server>
```

Verify:

```bash
oc whoami
```

Apply Argo Application:

```bash
oc apply -f argocd/sonarqube-application.yaml
```

Check:

```bash
oc get application sonarqube -n openshift-gitops
```

---

# 10. Initial Deployment Verification

Check namespace:

```bash
oc get ns sonarqube
```

Check all resources:

```bash
oc get all -n sonarqube
```

Check PVC:

```bash
oc get pvc -n sonarqube
```

Check Argo:

```bash
oc get application sonarqube -n openshift-gitops
```

---

# 11. Troubleshooting #1 — Argo CD Permission/RBAC Error

## Problem

Initial Argo CD sync failed because the Argo CD application-controller ServiceAccount could not create resources in the `sonarqube` namespace.

ServiceAccount:

```text
system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller
```

Typical failures included permissions to create:

```text
services
deployments
routes
PVCs
secrets
configmaps
```

## RCA

The Argo CD controller runs under its own ServiceAccount.

It does not automatically have arbitrary permissions in every application namespace.

The Application destination was:

```text
namespace: sonarqube
```

but the controller needed explicit permission in that namespace.

```
1. Create Role
cat <<'EOF' | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argocd-sonarqube-deployer
  namespace: sonarqube
rules:
  - apiGroups: [""]
    resources:
      - configmaps
      - services
      - persistentvolumeclaims
      - secrets
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete

  - apiGroups: ["apps"]
    resources:
      - deployments
      - replicasets
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete

  - apiGroups: ["route.openshift.io"]
    resources:
      - routes
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete

  - apiGroups: [""]
    resources:
      - pods
      - pods/log
    verbs:
      - get
      - list
      - watch
EOF
```

```
2. Bind the Role to Argo CD
cat <<'EOF' | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argocd-sonarqube-deployer
  namespace: sonarqube
subjects:
  - kind: ServiceAccount
    name: openshift-gitops-argocd-application-controller
    namespace: openshift-gitops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: argocd-sonarqube-deployer
EOF
```


---

# 12. Verify Argo CD Permissions

Check:

```bash
oc auth can-i create deployments \
  --as=system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller \
  -n sonarqube
```

Check Services:

```bash
oc auth can-i create services \
  --as=system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller \
  -n sonarqube
```

Check Routes:

```bash
oc auth can-i create routes \
  --as=system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller \
  -n sonarqube
```

Expected:

```text
yes
```

---

# 13. Argo CD RBAC Fix

A Role was created in:

```text
sonarqube
```

and bound to the Argo CD application-controller ServiceAccount.

The Role allowed the application-controller to manage the application resources required by this deployment.

Typical resources:

```text
configmaps
services
persistentvolumeclaims
secrets
deployments
replicasets
routes
pods
pods/log
```

Important lesson:

```text
Argo CD namespace != Application namespace
```

Argo CD may run in:

```text
openshift-gitops
```

while the application runs in:

```text
sonarqube
```

The controller therefore needs permissions in the destination namespace.

---

# 14. Troubleshooting #2 — PostgreSQL Permission Error

## Initial PostgreSQL image

Initially:

```text
postgres:16
```

was used.

The Pod failed during database initialization.

## Error

```text
chmod: changing permissions of '/var/lib/postgresql/data':
Operation not permitted

chmod: changing permissions of '/var/run/postgresql':
Operation not permitted

initdb: error: could not change permissions of directory
"/var/lib/postgresql/data": Operation not permitted
```

---

# 15. PostgreSQL RCA

OpenShift does not necessarily run containers as the fixed Docker UID/GID expected by traditional images.

Under the restricted SCC model, OpenShift commonly uses an arbitrary UID.

Example:

```text
1000780000
```

The standard PostgreSQL image attempted to change filesystem ownership/permissions on the mounted volume.

The OpenShift security model prevented those operations.

Therefore:

```text
PostgreSQL container
        |
        v
tries chmod/chown
        |
        v
OpenShift restricted SCC
        |
        v
Operation not permitted
```

---

# 16. Incorrect fsGroup Attempt

An attempt was made to use:

```yaml
securityContext:
  fsGroup: 999
```

OpenShift rejected it:

```text
.spec.securityContext.fsGroup:
Invalid value: []int64{999}: 999 is not an allowed group
```

## Lesson

Do not blindly use traditional Docker image UID/GID values such as:

```text
999
```

inside OpenShift restricted SCC.

The allowed security context is controlled by OpenShift SCC and namespace security policy.

---

# 17. PostgreSQL Final Fix

Use the OpenShift-compatible PostgreSQL image:

```text
quay.io/sclorg/postgresql-16-c9s:latest
```

Use the environment variables expected by this image:

```text
POSTGRESQL_USER
POSTGRESQL_PASSWORD
POSTGRESQL_DATABASE
```

The existing Secret keys can be mapped:

```yaml
- name: POSTGRESQL_USER
  valueFrom:
    secretKeyRef:
      name: sonarqube-postgres-secret
      key: POSTGRES_USER

- name: POSTGRESQL_PASSWORD
  valueFrom:
    secretKeyRef:
      name: sonarqube-postgres-secret
      key: POSTGRES_PASSWORD

- name: POSTGRESQL_DATABASE
  valueFrom:
    secretKeyRef:
      name: sonarqube-postgres-secret
      key: POSTGRES_DB
```

PostgreSQL data:

```text
/var/lib/pgsql/data
```

---

# 18. PostgreSQL Storage

PVC:

```text
sonarqube-postgres-pvc
```

Example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sonarqube-postgres-pvc
  namespace: sonarqube

spec:
  accessModes:
    - ReadWriteOnce

  storageClassName: lvms-vg1

  resources:
    requests:
      storage: 20Gi
```

Check:

```bash
oc get pvc -n sonarqube
```

Expected:

```text
sonarqube-postgres-pvc   Bound   20Gi   RWO   lvms-vg1
```

---

# 19. PostgreSQL Final Verification

```bash
oc get pod -n sonarqube
```

Expected:

```text
sonarqube-postgres-xxxxx   1/1   Running
```

Deployment:

```bash
oc get deployment sonarqube-postgres -n sonarqube
```

Expected:

```text
1/1
```

Service:

```bash
oc get svc sonarqube-postgres -n sonarqube
```

Expected:

```text
5432/TCP
```

---

# 20. Troubleshooting #3 — SonarQube ImagePullBackOff

## Initial image

The SonarQube Deployment initially used:

```yaml
image: sonarqube:2026.1-community
```

## Symptom

Pod:

```text
ImagePullBackOff
```

Example:

```text
pod/sonarqube-xxxxx   0/1   ImagePullBackOff
```

---

# 21. ImagePullBackOff Investigation

Do:

```bash
oc describe pod <sonarqube-pod> -n sonarqube
```

Look at:

```text
Events:
```

The important event was:

```text
Failed to pull image "sonarqube:2026.1-community"
```

and:

```text
manifest unknown
```

---

# 22. ImagePullBackOff RCA

This was not:

```text
DNS failure
```

not:

```text
registry authentication failure
```

and not:

```text
OpenShift networking failure
```

The registry was reachable.

The requested tag simply did not exist.

Key rule:

```text
manifest unknown
```

usually means:

```text
requested image/tag is not available
```

---

# 23. Correct Image Troubleshooting

For image problems always start with:

```bash
oc describe pod <pod> -n <namespace>
```

Do not immediately use:

```bash
oc logs <pod>
```

if the container has not started.

For an image-pull failure, container logs may not exist.

---

# 24. Temporary Image Test

A test Pod was created:

```bash
oc run sonarqube-image-test \
  -n sonarqube \
  --restart=Never \
  --image=docker.io/library/sonarqube:community \
  --command -- sleep 30
```

The PodSecurity warning generated for this temporary test was not itself the image pull failure.

After testing:

```bash
oc delete pod sonarqube-image-test -n sonarqube
```

---

# 25. Important Image Rule

Never randomly guess image tags.

For an image failure:

```text
1. Read Events
2. Identify exact registry
3. Identify exact repository
4. Identify exact tag
5. Validate tag
6. Test pull
7. Update Git
```

---

# 26. Troubleshooting #4 — SonarQube Pod Pending

After deleting the SonarQube Pod/PVC, the Deployment recreated the Pod.

However:

```text
sonarqube-xxxxx   0/1   Pending
```

---

# 27. Pending Pod Investigation

Command:

```bash
oc describe pod <sonarqube-pod> -n sonarqube
```

The important event was:

```text
FailedScheduling:
0/6 nodes are available:
persistentvolumeclaim "sonarqube-pvc" not found
```

---

# 28. Pending Pod RCA

The Deployment still contained:

```yaml
volumes:
  - name: sonarqube-storage
    persistentVolumeClaim:
      claimName: sonarqube-pvc
```

But the PVC had been deleted.

Therefore:

```text
Deployment
   |
   v
Pod
   |
   v
requires sonarqube-pvc
   |
   v
PVC does not exist
   |
   v
Scheduler cannot place Pod
   |
   v
Pending
```

---

# 29. PVC Fix

Recreate the PVC:

```bash
oc apply -f sonarqube/pvc.yaml
```

Verify:

```bash
oc get pvc -n sonarqube
```

Expected:

```text
sonarqube-pvc   Bound   20Gi   RWO   lvms-vg1
```

Then:

```bash
oc get pods -n sonarqube -w
```

Eventually:

```text
sonarqube-xxxxx   1/1   Running
```

---

# 30. Important PVC Lesson

Before deleting a PVC:

```bash
oc get deployment -n sonarqube
```

Check the Deployment:

```bash
oc get deployment sonarqube -n sonarqube -o yaml
```

Find:

```text
persistentVolumeClaim:
```

Never accidentally delete:

```text
sonarqube-postgres-pvc
```

when you only want to reset SonarQube application storage.

---

# 31. Troubleshooting #5 — Node Selector

SonarQube Deployment uses:

```yaml
nodeSelector:
  workload: sonarqube
```

If the Pod remains Pending, check:

```bash
oc get nodes -l workload=sonarqube
```

If nothing is returned:

```text
No resources found
```

then no node has the required label.

Check all labels:

```bash
oc get nodes --show-labels
```

Add label if required:

```bash
oc label node <worker-node> workload=sonarqube
```

Then:

```bash
oc get pods -n sonarqube -w
```

---

# 32. SonarQube Service

The final SonarQube Service is:

```text
Type: ClusterIP
Port: 9000
```

Example:

```text
service/sonarqube
ClusterIP
172.30.199.236
9000/TCP
```

This is correct when using an OpenShift Route.

Internal flow:

```text
SonarQube Route
      |
      v
Service sonarqube:9000
      |
      v
SonarQube Pod
```

---

# 33. Troubleshooting #6 — Route Exists in Git but Not in Cluster

The Git repository contained:

```text
sonarqube/route.yaml
```

and `route.yaml` was also included in:

```text
sonarqube/kustomization.yaml
```

However:

```bash
oc get route -n sonarqube
```

initially returned:

```text
No resources found
```

---

# 34. Check Argo Resource Status

Command:

```bash
oc get application sonarqube -n openshift-gitops \
  -o jsonpath='{range .status.resources[*]}{.kind}{"  "}{.name}{"  "}{.status}{"\n"}{end}'
```

The result showed:

```text
ConfigMap  sonarqube-config  Synced
Namespace  sonarqube  Synced
PersistentVolumeClaim  sonarqube-postgres-pvc  Synced
PersistentVolumeClaim  sonarqube-pvc  Synced
Service  sonarqube  Synced
Service  sonarqube-postgres  Synced
Deployment  sonarqube  Synced
Deployment  sonarqube-postgres  Synced
Route  sonarqube  OutOfSync
```

This was important.

It proved:

```text
Argo knows about the Route
```

but:

```text
Route was not successfully applied
```

---

# 35. Troubleshooting #7 — Route Host Permission

Argo Application condition showed:

```text
SyncError |
Failed last sync attempt:
Route.route.openshift.io "sonarqube" is invalid:
spec.host: Forbidden:
you do not have permission to set the host field of the route
```

---

# 36. Route Permission RCA

The Route initially contained:

```yaml
spec:
  host: sonarqube.ocp.lan
```

The Argo CD ServiceAccount had permission to create/manage the Route resource, but it was not allowed to claim that specific custom hostname.

Important distinction:

```text
Can create Route
```

does not necessarily mean:

```text
Can claim arbitrary Route host
```

OpenShift Route admission can restrict custom hostnames.

---

# 37. Route Fix

Instead of forcing:

```yaml
spec:
  host: sonarqube.ocp.lan
```

the explicit host was removed.

Final Route:

```yaml
apiVersion: route.openshift.io/v1
kind: Route

metadata:
  name: sonarqube
  namespace: sonarqube

spec:
  to:
    kind: Service
    name: sonarqube

  port:
    targetPort: http

  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

OpenShift then generated:

```text
sonarqube-sonarqube.apps.lab.ocp.lan
```

---

# 38. Final Route

Verification:

```bash
oc get route -n sonarqube
```

Final:

```text
NAME        HOST/PORT                              SERVICES    PORT   TERMINATION
sonarqube   sonarqube-sonarqube.apps.lab.ocp.lan   sonarqube   http   edge/Redirect
```

Get hostname:

```bash
oc get route sonarqube -n sonarqube \
  -o jsonpath='{.spec.host}{"\n"}'
```

Result:

```text
sonarqube-sonarqube.apps.lab.ocp.lan
```

---

# 39. Route Architecture

Final traffic flow:

```text
Browser
   |
   | HTTPS
   v
sonarqube-sonarqube.apps.lab.ocp.lan
   |
   v
192.168.22.1
   |
   v
OpenShift Router
   |
   v
Route
   |
   v
Service: sonarqube:9000
   |
   v
SonarQube Pod
```

---

# 40. Why ClusterIP Is Still Used

Do not confuse the Service with external access.

Service:

```text
ClusterIP
```

is internal.

Route:

```text
OpenShift Route
```

provides external HTTP/HTTPS access.

This is the preferred OpenShift architecture:

```text
Client
  |
  v
Route
  |
  v
ClusterIP Service
  |
  v
Pod
```

NodePort is not required for this design.

---

# 41. DNS Configuration

Lab DNS server:

```text
192.168.22.1
```

The Route hostname resolved as:

```text
sonarqube-sonarqube.apps.lab.ocp.lan
```

to:

```text
192.168.22.1
```

Verification:

```bash
nslookup sonarqube-sonarqube.apps.lab.ocp.lan
```

Expected:

```text
Name:
sonarqube-sonarqube.apps.lab.ocp.lan

Address:
192.168.22.1
```

---

# 42. DNS Verification

Also test:

```bash
getent hosts sonarqube-sonarqube.apps.lab.ocp.lan
```

Expected:

```text
192.168.22.1
sonarqube-sonarqube.apps.lab.ocp.lan
```

Ping:

```bash
ping sonarqube-sonarqube.apps.lab.ocp.lan
```

DNS resolution should point to:

```text
192.168.22.1
```

---

# 43. HTTPS Verification

From a machine that can reach the router:

```bash
curl -vk https://sonarqube-sonarqube.apps.lab.ocp.lan
```

`-k` is useful for a lab when the OpenShift router certificate is not trusted by the client.

For normal certificate validation:

```bash
curl -v https://sonarqube-sonarqube.apps.lab.ocp.lan
```

---

# 44. Browser Access

Open:

```text
https://sonarqube-sonarqube.apps.lab.ocp.lan
```

The Route uses:

```yaml
tls:
  termination: edge
```

Therefore TLS terminates at the OpenShift Router.

Traffic:

```text
Browser
  |
 HTTPS
  |
  v
OpenShift Router
  |
 HTTP
  |
  v
SonarQube Service :9000
```

---

# 45. Browser Works From Terminal But Not Chrome

If:

```bash
curl -vk https://sonarqube-sonarqube.apps.lab.ocp.lan
```

works but Chrome does not, troubleshoot the client/browser.

From the same client machine:

```bash
getent hosts sonarqube-sonarqube.apps.lab.ocp.lan
```

Then:

```bash
curl -vk https://sonarqube-sonarqube.apps.lab.ocp.lan
```

Check:

```text
DNS
Proxy
Certificate
Firewall
Network routing
Chrome proxy settings
```

Common Chrome errors:

```text
ERR_NAME_NOT_RESOLVED
    -> DNS problem

ERR_CONNECTION_REFUSED
    -> listener/network/service problem

ERR_CONNECTION_TIMED_OUT
    -> routing/firewall/network problem

ERR_CERT_AUTHORITY_INVALID
    -> certificate trust problem

ERR_PROXY_CONNECTION_FAILED
    -> proxy problem
```

---

# 46. End-to-End Verification

## Step 1 — Pods

```bash
oc get pods -n sonarqube -o wide
```

Expected:

```text
sonarqube-xxxxx            1/1   Running
sonarqube-postgres-xxxxx   1/1   Running
```

---

## Step 2 — Deployments

```bash
oc get deployment -n sonarqube
```

Expected:

```text
sonarqube            1/1
sonarqube-postgres   1/1
```

---

## Step 3 — Services

```bash
oc get svc -n sonarqube
```

Expected:

```text
sonarqube            ClusterIP   ...   9000/TCP
sonarqube-postgres   ClusterIP   ...   5432/TCP
```

---

## Step 4 — PVCs

```bash
oc get pvc -n sonarqube
```

Expected:

```text
sonarqube-pvc
sonarqube-postgres-pvc
```

Both:

```text
Bound
```

---

## Step 5 — Route

```bash
oc get route -n sonarqube
```

Expected:

```text
sonarqube-sonarqube.apps.lab.ocp.lan
```

---

## Step 6 — Argo CD

```bash
oc get application sonarqube -n openshift-gitops
```

Expected:

```text
Synced
Healthy
```

---

# 47. Argo Resource Verification

Use:

```bash
oc get application sonarqube -n openshift-gitops \
  -o jsonpath='{range .status.resources[*]}{.kind}{"  "}{.name}{"  "}{.status}{"\n"}{end}'
```

Expected:

```text
ConfigMap              sonarqube-config          Synced
Namespace              sonarqube                 Synced
PersistentVolumeClaim  sonarqube-postgres-pvc    Synced
PersistentVolumeClaim  sonarqube-pvc             Synced
Service                sonarqube                 Synced
Service                sonarqube-postgres        Synced
Deployment             sonarqube                 Synced
Deployment             sonarqube-postgres        Synced
Route                  sonarqube                 Synced
```

---

# 48. Useful Troubleshooting Commands

## Everything

```bash
oc get all -n sonarqube
```

## Pods

```bash
oc get pods -n sonarqube -o wide
```

## Pod details

```bash
oc describe pod <pod-name> -n sonarqube
```

## Pod logs

```bash
oc logs <pod-name> -n sonarqube
```

## Deployment logs

```bash
oc logs deployment/sonarqube -n sonarqube
```

```bash
oc logs deployment/sonarqube-postgres -n sonarqube
```

## PVC

```bash
oc get pvc -n sonarqube
```

## Service

```bash
oc get svc -n sonarqube
```

## Route

```bash
oc get route -n sonarqube
```

## Endpoints

```bash
oc get endpoints -n sonarqube
```

## EndpointSlices

```bash
oc get endpointslice -n sonarqube
```

## Argo Application

```bash
oc get application sonarqube -n openshift-gitops
```

## Argo conditions

```bash
oc get application sonarqube -n openshift-gitops \
  -o jsonpath='{range .status.conditions[*]}{.type}{" | "}{.message}{"\n"}{end}'
```

---

# 49. L2/L3 Troubleshooting Decision Tree

```text
User cannot access SonarQube
          |
          v
Is Pod Running?
          |
     +----+----+
     |         |
    NO        YES
     |         |
     v         v
describe     Is Pod Ready?
pod              |
     |       +---+---+
     |       |       |
     v      NO      YES
Check         |       |
Events        v       v
              Logs   Service
              Probe    |
              JDBC     v
                     Route
                       |
                 +-----+-----+
                 |           |
                NO          YES
                 |           |
                 v           v
             Route/RBAC    DNS
             admission       |
                             v
                          Browser
                             |
                     +-------+-------+
                     |               |
                   Works          Doesn't
                     |               |
                     v               v
                   Done        DNS/Proxy/
                               Certificate/
                               Firewall
```

---

# 50. Failure-to-RCA Quick Reference

| Symptom                                    | RCA                                    | Command                          | Fix                                            |
| ------------------------------------------ | -------------------------------------- | -------------------------------- | ---------------------------------------------- |
| Argo cannot create Deployment              | RBAC                                   | `oc auth can-i`                  | Role/RoleBinding                               |
| Argo cannot create Route                   | RBAC/admission                         | `oc auth can-i` + Argo condition | Fix permissions                                |
| `spec.host Forbidden`                      | Custom Route host not permitted        | Argo condition                   | Remove `spec.host` or configure host admission |
| PostgreSQL `chmod Operation not permitted` | OpenShift arbitrary UID/security model | `oc logs`                        | OpenShift-compatible PostgreSQL image          |
| `fsGroup 999 not allowed`                  | SCC rejects group                      | `oc describe pod`                | Don't force traditional GID                    |
| `ImagePullBackOff`                         | Image/tag problem                      | `oc describe pod`                | Validate image/tag                             |
| `manifest unknown`                         | Image tag does not exist               | Pod Events                       | Use valid image/tag                            |
| Pod `Pending`                              | PVC missing                            | `oc describe pod`                | Recreate PVC                                   |
| Pod `Pending`                              | Node selector mismatch                 | `oc get nodes -l`                | Add correct node label                         |
| Route missing                              | Route not rendered/applied             | Argo resource status             | Check Kustomization/Argo                       |
| DNS failure                                | DNS record/client resolver             | `nslookup`                       | Fix DNS                                        |
| Curl works, Chrome fails                   | Browser-side issue                     | `curl`, Chrome error             | Proxy/cert/DNS                                 |

---

# 51. Important OpenShift Lessons

## Lesson 1 — Always Check Events

For Kubernetes/OpenShift troubleshooting:

```bash
oc describe pod <pod>
```

is often the fastest RCA command.

Especially for:

```text
Pending
ImagePullBackOff
ErrImagePull
ContainerCreating
CrashLoopBackOff
```

---

## Lesson 2 — Understand OpenShift Security

Traditional Docker assumptions about:

```text
UID
GID
fsGroup
chmod
chown
```

may fail under OpenShift SCC.

Always consider:

```text
SCC
arbitrary UID
filesystem permissions
container image compatibility
```

---

## Lesson 3 — PVC Is Part of Application Availability

A Deployment can exist while its Pod cannot start because:

```text
PVC missing
PVC Pending
PVC unavailable
storage provisioning failed
```

Therefore:

```bash
oc get pvc
```

should be part of every stateful application troubleshooting workflow.

---

## Lesson 4 — Route Is More Than RBAC

A user may have permission to create a Route but still be prohibited from using a particular:

```text
spec.host
```

because OpenShift Route admission can restrict custom hostnames.

---

## Lesson 5 — GitOps Source of Truth

The intended architecture is:

```text
Git
 |
 v
Argo CD
 |
 v
OpenShift
```

Do not permanently fix resources using:

```bash
oc edit
```

or:

```bash
oc patch
```

without committing the desired configuration back to Git.

Because:

```yaml
selfHeal: true
```

means Argo can revert manual changes.

---

# 52. Fresh Deployment Checklist

When deploying again from scratch, follow this order.

## Step 1 — Verify cluster

```bash
oc get nodes
```

All required nodes should be:

```text
Ready
```

---

## Step 2 — Verify storage

```bash
oc get storageclass
```

Confirm:

```text
lvms-vg1
```

---

## Step 3 — Verify Argo CD

```bash
oc get pods -n openshift-gitops
```

Confirm Argo components are healthy.

---

## Step 4 — Verify Git

```bash
git clone https://github.com/rakesh611/sonarqube-gitops.git
cd sonarqube-gitops
```

Check:

```bash
cat kustomization.yaml
cat sonarqube/kustomization.yaml
cat postgres/kustomization.yaml
```

Confirm:

```text
route.yaml
```

is included.

---

## Step 5 — Check Argo RBAC

```bash
oc auth can-i create deployments \
  --as=system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller \
  -n sonarqube
```

```bash
oc auth can-i create routes \
  --as=system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller \
  -n sonarqube
```

Expected:

```text
yes
```

---

## Step 6 — Apply Argo Application

```bash
oc apply -f argocd/sonarqube-application.yaml
```

---

## Step 7 — Monitor Argo

```bash
oc get application sonarqube -n openshift-gitops -w
```

---

## Step 8 — Monitor PVC

```bash
oc get pvc -n sonarqube -w
```

---

## Step 9 — Monitor Pods

```bash
oc get pods -n sonarqube -w
```

---

## Step 10 — Check PostgreSQL First

```bash
oc get pod -n sonarqube
```

PostgreSQL must become:

```text
1/1 Running
```

---

## Step 11 — Check SonarQube

```bash
oc get pod -n sonarqube
```

SonarQube must become:

```text
1/1 Running
```

---

## Step 12 — Check Services

```bash
oc get svc -n sonarqube
```

---

## Step 13 — Check Route

```bash
oc get route -n sonarqube
```

---

## Step 14 — Check DNS

```bash
nslookup sonarqube-sonarqube.apps.lab.ocp.lan
```

---

## Step 15 — Test HTTPS

```bash
curl -vk https://sonarqube-sonarqube.apps.lab.ocp.lan
```

---

## Step 16 — Browser

Open:

```text
https://sonarqube-sonarqube.apps.lab.ocp.lan
```

---

# 53. Final Healthy State

Final deployment should look like:

```text
                         GitHub
                           |
                           v
                         Argo CD
                           |
                           v
                    OpenShift Cluster
                           |
                    Namespace sonarqube
                           |
             +-------------+-------------+
             |                           |
             v                           v
        SonarQube                   PostgreSQL
          1/1                          1/1
        :9000                          :5432
             |                           |
             +-------------+-------------+
                           |
                          PVC
                           |
                       lvms-vg1
                           |
                           v
                    OpenShift Route
                           |
                           v
        sonarqube-sonarqube.apps.lab.ocp.lan
                           |
                           v
                      DNS / Router
                           |
                           v
                       Browser
```

---

# 54. Final Verification Commands

Run these commands after every deployment:

```bash
oc get all -n sonarqube
```

```bash
oc get pvc -n sonarqube
```

```bash
oc get route -n sonarqube
```

```bash
oc get application sonarqube -n openshift-gitops
```

```bash
oc get application sonarqube -n openshift-gitops \
  -o jsonpath='{range .status.resources[*]}{.kind}{"  "}{.name}{"  "}{.status}{"\n"}{end}'
```

```bash
curl -vk https://$(oc get route sonarqube -n sonarqube -o jsonpath='{.spec.host}')
```

---

# 55. Security Note

The lab repository contains Kubernetes Secret manifests.

Do not use real production credentials in a public Git repository.

For production use one of:

```text
Sealed Secrets
External Secrets Operator
HashiCorp Vault
Cloud Secret Manager
Enterprise secret-management solution
```

Any password that has been exposed in a public repository should be rotated before production use.

---

# 56. One-Page Troubleshooting Cheat Sheet

```bash
# Cluster
oc get nodes

# Namespace
oc get ns sonarqube

# Everything
oc get all -n sonarqube

# Pods
oc get pods -n sonarqube -o wide

# Pod RCA
oc describe pod <pod> -n sonarqube

# Logs
oc logs <pod> -n sonarqube

# PVC
oc get pvc -n sonarqube

# Storage
oc get storageclass

# Services
oc get svc -n sonarqube

# Endpoints
oc get endpoints -n sonarqube

# Route
oc get route -n sonarqube

# DNS
nslookup sonarqube-sonarqube.apps.lab.ocp.lan

# HTTPS
curl -vk https://sonarqube-sonarqube.apps.lab.ocp.lan

# Argo
oc get application sonarqube -n openshift-gitops

# Argo resource status
oc get application sonarqube -n openshift-gitops \
  -o jsonpath='{range .status.resources[*]}{.kind}{"  "}{.name}{"  "}{.status}{"\n"}{end}'

# Argo conditions
oc get application sonarqube -n openshift-gitops \
  -o jsonpath='{range .status.conditions[*]}{.type}{" | "}{.message}{"\n"}{end}'

# Argo Route permission
oc auth can-i create routes \
  --as=system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller \
  -n sonarqube
```

---

# 57. Deployment Status Achieved

At the end of this deployment/troubleshooting exercise:

```text
SonarQube Pod       -> 1/1 Running
PostgreSQL Pod      -> 1/1 Running
SonarQube Service   -> ClusterIP :9000
PostgreSQL Service  -> ClusterIP :5432
PVCs                -> Bound
Argo CD             -> Managing application
Route               -> Created
Route TLS           -> edge/Redirect
DNS                 -> Resolving to 192.168.22.1
Route hostname      -> sonarqube-sonarqube.apps.lab.ocp.lan
```

The major issues resolved were:

```text
1. Argo CD RBAC permission failure
2. PostgreSQL OpenShift filesystem permission failure
3. Invalid PostgreSQL fsGroup assumption
4. Invalid SonarQube image tag
5. Missing SonarQube PVC
6. Pod Pending due to missing PVC
7. Route OutOfSync
8. Route custom-host permission failure
9. DNS validation
10. Browser/client-side access troubleshooting
```

This README is the baseline runbook for the next SonarQube OpenShift deployment.
