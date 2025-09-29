# K8s_Troubleshooting_Guide

This repository provides a comprehensive guide to troubleshooting Kubernetes applications, focusing on debugging a two-tier application (web frontend and MySQL database) and control plane issues. It covers steps to diagnose connectivity, pod issues, service configurations, environment variables, control plane components, and deployment problems, along with namespace management and autocompletion setup.

## 1. Troubleshooting Application Failure

### Scenario
You have a two-tier application in Kubernetes:
- **Pods**:
  - `mysql`: Runs on port 3306.
  - `webapp-mysql`: Runs on port 8080.
- **Services**:
  - `mysql-service`: Exposes port 3306.
  - `web-service`: Exposes port 8080, with a NodePort of 30081 for external access.

The application is failing, and you need to troubleshoot accessibility and functionality.

### Steps to Troubleshoot

1. **Check Frontend Accessibility**:
   Start by testing the web frontend using the service's NodePort:
   ```bash
   curl http://<node-ip>:30081
   ```
   Replace `<node-ip>` with the cluster node IP where the service is exposed.

2. **Inspect the Web Service**:
   Verify the service configuration, selector, and endpoints:
   ```bash
   kubectl describe service web-service
   ```
   - **Selector**: Ensure it matches the pod's label (e.g., `app=webapp-mysql`).
   - **Endpoints**: Confirm an endpoint exists (e.g., `10.32.0.6:8080`). If `<none>`, the service isn’t detecting the pod, likely due to a mismatched selector or label.

3. **Check the Web Pod**:
   Inspect the pod’s status and logs:
   ```bash
   kubectl get pod -o wide
   kubectl describe pod -l app=webapp-mysql
   ```
   - Check the pod’s status (e.g., `Running`, `CrashLoopBackOff`).
   - Note the number of restarts, indicating crashes.
   - View logs:
     ```bash
     kubectl logs -l app=webapp-mysql
     ```
   - For previous pod instances (if restarted):
     ```bash
     kubectl logs -l app=webapp-mysql --previous
     ```

4. **Inspect the Database Service**:
   Similar to the web service, check the MySQL service:
   ```bash
   kubectl describe service mysql-service
   ```
   - Verify the selector (e.g., `app=mysql`) and endpoint (e.g., `<mysql-pod-ip>:3306`).
   - If endpoints show `<none>`, check for mismatched labels between the service and pod.

5. **Check the Database Pod**:
   Inspect the MySQL pod:
   ```bash
   kubectl get pod -l app=mysql -o wide
   kubectl describe pod -l app=mysql
   kubectl logs -l app=mysql
   ```
   - Ensure the pod is running without excessive restarts.
   - Check logs for errors (e.g., authentication issues).

6. **Validate Configuration**:
   - **Database Credentials**: Verify the database user and password in the pod’s environment variables or ConfigMap.
   - **Environment Variables**: Ensure the `DB_HOST` environment variable in the web pod matches the `mysql-service` name (e.g., `mysql-service`).
   - **Port Matching**: Confirm the service port (3306) matches the MySQL pod’s port.
   - **Selector Labels**: Ensure the `mysql-service` selector matches the labels defined in the MySQL pod’s metadata (e.g., `app=mysql`).
   - **Endpoints**: Verify the service endpoint IP and port align with the MySQL pod’s IP and port.

7. **Fix Common Issues**:
   - **Mismatched Selectors**: Update the service or pod labels to match.
     Example pod label in `metadata`:
     ```yaml
     labels:
       app: mysql
     ```
     Example service selector:
     ```yaml
     selector:
       app: mysql
     ```
   - **Incorrect Environment Variables**: Update the ConfigMap or pod spec:
     ```bash
     kubectl edit configmap <configmap-name>
     ```
     Ensure `DB_HOST` is set to `mysql-service`.
   - **Pod Restart Loops**: Check logs for errors (e.g., missing database, wrong credentials) and fix the underlying issue.

8. **Apply Fixes**:
   If manual changes are needed, update the YAML file and reapply:
   ```bash
   kubectl replace --force -f /path/to/filename.yaml
   ```

## 2. Troubleshooting Control Plane Failure

### Issue
The Kubernetes control plane is malfunctioning, affecting cluster operations.

### Steps to Troubleshoot

1. **Check Node Status**:
   Verify the health of cluster nodes:
   ```bash
   kubectl get nodes
   ```
   Ensure all nodes are in the `Ready` state.

2. **Inspect Control Plane Pods**:
   Check the status of control plane components running in the `kube-system` namespace:
   ```bash
   kubectl get pods -n kube-system
   ```
   Look for pods like `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, and `kube-proxy`.

3. **Verify Control Plane Services** (if deployed as services on the host):
   If control plane components are running as system services, check their status:
   ```bash
   sudo systemctl status kube-apiserver
   sudo systemctl status kube-controller-manager
   sudo systemctl status kube-scheduler
   sudo systemctl status kubelet
   sudo systemctl status kube-proxy
   ```

4. **Check Logs**:
   - For control plane pods:
     ```bash
     kubectl logs kube-apiserver-<node-name> -n kube-system
     kubectl logs kube-controller-manager-<node-name> -n kube-system
     kubectl logs kube-scheduler-<node-name> -n kube-system
     ```
   - For system services:
     ```bash
     sudo journalctl -u kube-apiserver
     sudo journalctl -u kube-controller-manager
     sudo journalctl -u kube-scheduler
     ```

5. **Common Issues**:
   - **CA Certificate Error**: If logs show errors like `ca.crt not found`, verify the certificate path (e.g., `/etc/kubernetes/pki/ca.crt`) is correctly mounted:
     ```bash
     ls -l /etc/kubernetes/pki/
     ```
   - **Manifest Path**: Ensure control plane manifests exist:
     ```bash
     ls -l /etc/kubernetes/manifests/
     ```
     Fix missing or incorrect paths as needed.

6. **Further Resources**:
   Refer to the official Kubernetes documentation for detailed debugging:
   - [Debugging a Kubernetes Cluster](https://kubernetes.io/docs/tasks/debug/debug-cluster/)

### Deployment Not Running with Expected Pods

#### Issue
A deployment (e.g., `webapp-mysql`) does not create the expected number of pods or pods are in an unexpected state (e.g., `Pending`).

#### Steps to Troubleshoot
1. **Check Deployment Events**:
   ```bash
   kubectl describe deployment webapp-mysql
   ```
   Look for errors in scaling or pod creation.

2. **Inspect ReplicaSet**:
   ```bash
   kubectl describe replicaset -l app=webapp-mysql
   ```
   Verify the ReplicaSet is managing the correct number of pods.

3. **Check Pod Status**:
   ```bash
   kubectl describe pod -l app=webapp-mysql
   ```
   If pods are in `Pending` state, the `kube-scheduler` may have failed to assign a node.

4. **Debug Kube-Scheduler**:
   Check scheduler logs:
   ```bash
   kubectl describe pod kube-scheduler-<node-name> -n kube-system
   kubectl logs kube-scheduler-<node-name> -n kube-system
   ```
   Look for scheduling errors (e.g., resource constraints, taints/tolerations).

5. **Scaling Issues**:
   If scaling (e.g., `kubectl scale deployment webapp-mysql --replicas=3`) doesn’t increase pod count, the `kube-controller-manager` may be failing:
   ```bash
   kubectl describe pod kube-controller-manager-<node-name> -n kube-system
   kubectl logs kube-controller-manager-<node-name> -n kube-system
   ```
   - Check for errors like `file not found` for `controller-manager.conf`.
   - Verify the configuration path (e.g., `/etc/kubernetes/controller-manager.conf`).

6. **Fix Configuration**:
   Update or correct paths in the control plane configuration files and restart affected services or pods.

## Changing the Default Namespace

To troubleshoot in a specific namespace (e.g., `alpha`):
```bash
kubectl config set-context --current --namespace=alpha
```

Verify resources in the namespace:
```bash
kubectl describe deployment webapp-mysql
kubectl describe service mysql-service
```

## Setting Up Autocompletion

For easier command-line interaction, enable `kubectl` autocompletion:
```bash
alias k=kubectl
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
```
Add these to your `~/.bashrc` or equivalent for persistence.

## Example Workflow
1. **Application Failure**:
   - Test web service accessibility:
     ```bash
     curl http://<node-ip>:30081
     ```
   - If it fails, check service and pod details:
     ```bash
     kubectl describe service web-service
     kubectl describe pod -l app=webapp-mysql
     kubectl logs -l app=webapp-mysql
     ```
   - Move to the database service and pod:
     ```bash
     kubectl describe service mysql-service
     kubectl describe pod -l app=mysql
     kubectl logs -l app=mysql
     ```
   - Fix issues like mismatched selectors, incorrect `DB_HOST`, or missing credentials in the ConfigMap.

2. **Control Plane Failure**:
   - Check node and pod status:
     ```bash
     kubectl get nodes
     kubectl get pods -n kube-system
     ```
   - Inspect logs for control plane components and fix certificate or manifest path issues.

3. **Deployment Issues**:
   - Verify deployment, ReplicaSet, and pod events:
     ```bash
     kubectl describe deployment webapp-mysql
     kubectl describe replicaset -l app=webapp-mysql
     kubectl describe pod -l app=webapp-mysql
     ```
   - Debug scheduler or controller-manager logs if pods are pending or scaling fails.

## Conclusion
Troubleshooting Kubernetes applications and control plane issues requires a systematic approach, from checking service accessibility and pod logs to verifying control plane components and deployment events. By validating configurations, selectors, environment variables, and control plane health, you can diagnose and resolve issues effectively. Tools like autocompletion and official documentation further streamline the process for managing complex Kubernetes clusters.
