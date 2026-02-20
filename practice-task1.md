# OpenShift Practical Practice Tasks (CLI + GUI)

This lab is designed for **hands-on practice** of core OpenShift operations using **both CLI and Web Console (GUI)**.
Focus areas: Projects, Pods, Deployments, Replicas, `oc get`, `oc describe`, `oc explain`, Events, `oc create`, and `oc apply`.

---

## Prerequisites

* Access to an OpenShift cluster (CRC / SNO / Dev Sandbox)
* Logged in using CLI:

```bash
oc whoami
```

* Access to OpenShift Web Console

---

## Task 1: Create a New Project (Namespace)

### CLI

```bash
oc new-project gui-cli-practice
oc project
oc get projects
```

### GUI

1. Login to OpenShift Web Console
2. Go to **Projects**
3. Click **Create Project**
4. Enter:

   * Name: `gui-cli-practice`
   * Display Name: `GUI CLI Practice`
5. Create and switch to the project

---

## Task 2: Working with Pods

### CLI – Create Pod

```bash
oc run test-pod \
  --image=registry.access.redhat.com/ubi9/ubi \
  --command -- sleep 3600
```

Verify:

```bash
oc get pods
oc describe pod test-pod
```

### GUI – Inspect Pod

1. Go to **Workloads → Pods**
2. Click `test-pod`
3. Check:

   * Status
   * Node
   * Events
   * Logs
   * YAML

---

## Task 3: Create a Deployment

### CLI – Imperative

```bash
oc create deployment web-deploy \
  --image=registry.access.redhat.com/ubi9/nginx-120
```

Verify:

```bash
oc get deployment
oc get pods
```

### GUI – Web Console

1. Go to **+Add**
2. Select **Container Images**
3. Image: `registry.access.redhat.com/ubi9/nginx-120`
4. Application Name: `web-gui`
5. Resource Type: **Deployment**
6. Click **Create**

Verify in **Topology View**.

---

## Task 4: Replicas and Scaling

### CLI

```bash
oc scale deployment web-deploy --replicas=3
oc get pods
```

### GUI

1. Go to **Workloads → Deployments**
2. Select `web-gui`
3. Increase replicas to `3`
4. Save and observe pod creation

---

## Task 5: Get, Describe, and Events

### CLI

```bash
oc get all
oc describe deployment web-deploy
oc get events
oc get events --sort-by='.lastTimestamp'
```

### GUI

1. Go to **Workloads → Pods**
2. Select any pod
3. Open **Events** and **YAML** tabs
4. Delete one pod and observe auto-recreation

---

## Task 6: oc explain (Exam-Critical)

```bash
oc explain pod
oc explain pod.spec
oc explain pod.spec.containers
oc explain deployment.spec
oc explain deployment.spec --recursive
```

**Practice:** Identify where the following are defined:

* image
* replicas
* env variables

---

## Task 7: oc create vs oc apply

### Generate YAML

```bash
oc create deployment yaml-app \
  --image=registry.access.redhat.com/ubi9/nginx-120 \
  --dry-run=client -o yaml > deploy.yaml
```

### Create Resource

```bash
oc create -f deploy.yaml
oc get deployment yaml-app
```

### Modify YAML

Edit `deploy.yaml`:

```yaml
spec:
  replicas: 3
```

### Apply Changes

```bash
oc apply -f deploy.yaml
oc get deployment yaml-app
```

**Observation:**

* What happens if `oc create` is run again?
* Why is `oc apply` preferred for updates?

---

## Task 8: Edit Resources

### CLI

```bash
oc edit deployment yaml-app
```

### GUI

1. Go to **Deployments → yaml-app**
2. Open **YAML**
3. Modify replicas or image
4. Save and observe rollout

---

## Task 9: Cleanup

### CLI

```bash
oc delete pod test-pod
oc delete deployment web-deploy yaml-app
oc delete project gui-cli-practice
```

### GUI

* Delete deployment `web-gui`
* Confirm pods are removed

---

## Summary

You practiced:

* Projects / Namespaces
* Pods and Deployments
* Replicas and Scaling
* `oc get`, `oc describe`, `oc explain`
* Events and troubleshooting
* `oc create` vs `oc apply`
* CLI and GUI workflows

---

**End of Lab**
