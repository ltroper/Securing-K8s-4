# Securing LKE Cluster Part 4: Configuring Security Context for a Pod

## Introduction

Securing workloads in a Kubernetes (K8s) cluster goes beyond network policies and RBAC. A crucial layer in the security model is the **security context** for Pods, which allows you to define privileges and access control settings at the container level. This ensures that your containers are running with the least privilege necessary, preventing unauthorized access or privilege escalation.

In this guide, we will:
- Understand how to set up a **security context** for a Pod.
- Compare Pods with and without security contexts.
- Learn why configuring a security context is vital for ensuring the safety of your applications.

By the end, you'll be familiar with how security contexts work and how they can mitigate potential risks such as privilege escalation.

---

## Step 1: Create a Pod with a Security Context

First, let’s define a pod that specifies a **security context**. The security context will control which user and group the container runs as and prevent privilege escalation within the container.

Create the following YAML file `security-context-demo.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000      # The container runs as user 1000
    runAsGroup: 3000     # The primary group is set to 3000
    fsGroup: 2000        # File system group is set to 2000 for mounted volumes
    supplementalGroups: [4000]  # Additional groups assigned
  volumes:
  - name: sec-ctx-vol
    emptyDir: {}
  containers:
  - name: sec-ctx-demo
    image: busybox:1.28
    command: [ "sh", "-c", "sleep 1h" ]  # Keeps the container running for 1 hour
    volumeMounts:
    - name: sec-ctx-vol
      mountPath: /data/demo
    securityContext:
      allowPrivilegeEscalation: false   # Prevents privilege escalation inside the container
```

This Pod is configured with:
- `runAsUser`: Forces the container to run as user ID `1000`.
- `runAsGroup`: Sets the primary group to `3000`.
- `fsGroup`: Sets file system permissions for volumes.
- `allowPrivilegeEscalation`: Ensures the container can't elevate its privileges (e.g., by running `sudo`).

---

## Step 2: Create a Pod Without a Security Context

Next, define a similar pod but **without a security context**, allowing it to run with the default root privileges.

Create the following YAML file `no-security-context-demo.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-security-context-demo
spec:
  volumes:
  - name: sec-ctx-vol
    emptyDir: {}
  containers:
  - name: sec-ctx-demo
    image: busybox:1.28
    command: [ "sh", "-c", "sleep 1h" ]  # Keeps the container running for 1 hour
    volumeMounts:
    - name: sec-ctx-vol
      mountPath: /data/demo2
```

In this case, the container will run with **root** privileges by default, and no specific users, groups, or file system permissions are enforced.

---

## Step 3: Deploy Both Pods

Deploy both Pods to your Kubernetes cluster using `kubectl`:

```bash
kubectl apply -f security-context-demo.yaml
kubectl apply -f no-security-context-demo.yaml
```

Verify the Pods are running:

```bash
kubectl get pods
```

---

## Step 4: Compare Security Behavior

### 1. **Exec into the Pod without a Security Context**:

```bash
kubectl exec -ti no-security-context-demo -- sh
```

Inside the container, run the following commands to observe the security behavior:

- **Check the running processes**:
    ```bash
    ps
    ```
    You’ll notice that **everything runs as root**, which is a security risk, especially if the application is compromised.

- **Check the permissions on the mounted volume**:
    ```bash
    cd /data
    ls -l
    ```
    Files and directories are owned by **root**, indicating that the container is running with full root privileges, increasing the attack surface.

- **Check the user ID**:
    ```bash
    id
    ```
    The output will show that the container is running as **UID 0 (root)**, with all processes having full access to system resources.

**Why is this dangerous?**

- **Privilege Escalation**: A malicious actor gaining access to this container can easily escalate privileges, gaining control over the host system.
- **Lack of Isolation**: Running as root reduces the isolation between the container and the host, potentially leading to system compromise if vulnerabilities are exploited.

---

### 2. **Exec into the Pod with a Security Context**:

```bash
kubectl exec -ti security-context-demo -- sh
```

Run the same commands and observe the behavior:

- **Check the running processes**:
    ```bash
    ps
    ```
    This time, the processes are not running as root but as the **non-root user 1000**, reducing the risk of privilege escalation.

- **Check the permissions on the mounted volume**:
    ```bash
    cd /data
    ls -l
    ```
    Files and directories are owned by **UID 1000** and **GID 2000**, enforcing the user and group settings defined in the security context.

- **Check the user ID**:
    ```bash
    id
    ```
    The output will show **UID 1000** and **GID 3000**, indicating the container runs under a restricted user and group, providing an additional layer of security.

**Why is this safer?**

- **Least Privilege**: The container runs as a non-root user, following the principle of least privilege, significantly limiting what an attacker could do if they gain access.
- **File System Security**: The file system is set up with non-root ownership, reducing the risk of unauthorized access or tampering with sensitive files.
- **No Privilege Escalation**: With `allowPrivilegeEscalation: false`, even if an attacker tries to elevate their privileges, they will be blocked.

---

## Conclusion

By using **security contexts** in Kubernetes, you can enforce critical security settings at the container level, ensuring that your workloads run with the least privilege necessary. In this example, we saw how running a container with no security context left it vulnerable, while a properly configured security context mitigated these risks.

As you continue to secure your Kubernetes clusters, always apply security best practices such as:
- Running containers as non-root users.
- Disabling privilege escalation.
- Controlling access to volumes and sensitive file systems.

This guide concludes Part 4 of securing LKE clusters. Continue reinforcing your Kubernetes security posture by configuring security contexts for all your pods and integrating them with other security tools like Tracee (as discussed in Part 3).
