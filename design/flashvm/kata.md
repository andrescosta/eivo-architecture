Perfect. Running a fully matching v1.32.0 control plane and node ecosystem makes this even simpler. You do not need to alter any of your orchestration manifests or wrestle with version translation mismatches.

Because you are using Kustomize v5.5.0, you can keep the infrastructure scripts entirely clean and drive the whole setup declaratively.

---

### 1. The Script Addition (Host Level)

Since you are on Debian 12, just drop this exact snippet into your existing host initialization block right before joining nodes to the cluster. This makes sure KVM is active and grabs `kata-deploy` to extract the correct static binary artifacts directly into `/opt/kata`:

```bash
#!/usr/bin/env bash
# Ensure KVM is up on the Debian host
sudo modprobe kvm_intel || sudo modprobe kvm_amd
sudo chmod 666 /dev/kvm

# Deploy Kata v3 components directly onto the cluster nodes
kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/kata-rbac/base/kata-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/kata-deploy/base/kata-deploy.yaml

```

---

### 2. The Kustomize Layer (Cluster Level)

Instead of executing raw `kubectl` edits or manual configs, wrap your sandbox definition straight into your existing Kustomize overlay directory.

Create a `runtime-class.yaml` component inside your project structure:

```yaml
# runtime-class.yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-fc
handler: kata-fc

```

Then reference it directly in your `kustomization.yaml`:

```yaml
# kustomization.yaml
resources:
  - runtime-class.yaml

```

---

### 3. Execution Schema (User Projects)

When your backend generates a manifest to execute a user's code block, it will output a minimal pod target definition that matches your v1.32 specification perfectly:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-user-job
spec:
  runtimeClassName: kata-fc   # Tells k8s 1.32 to bypass runc and hand off to Firecracker
  containers:
  - name: sandbox-runtime
    image: python:3.11-alpine
    command: ["python", "-c", "print('Executed cleanly inside Firecracker microVM')"]
    volumeMounts:
    - name: scratch-space
      mountPath: /tmp
  volumes:
  - name: scratch-space
    emptyDir: {}               # Zero-configuration workspace storage mapping

```

This ensures your core services run unmodified on the default container engine, while user-provided execution steps are wrapped within a secure hardware isolation boundary on any node they happen to hit. Simple, scriptable, and done.