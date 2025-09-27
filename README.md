# K8s_Troubleshooting_Guide

This repository provides a comprehensive guide to troubleshooting Kubernetes applications, focusing on debugging a two-tier application with a web frontend and a MySQL database. It covers steps to diagnose connectivity, pod issues, service configurations, and environment variables, along with namespace management.

## Troubleshooting Application Failure

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
   curl http://<web-service-ip>:30081
   ```
   Replace `<web-service-ip>` with the cluster IP or node IP where the service is exposed.

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

## Example Workflow
1. Start with the web service accessibility:
   ```bash
   curl http://<node-ip>:30081
   ```
2. If it fails, check the service and pod details:
   ```bash
   kubectl describe service web-service
   kubectl describe pod -l app=webapp-mysql
   kubectl logs -l app=webapp-mysql
   ```
3. Move to the database service and pod:
   ```bash
   kubectl describe service mysql-service
   kubectl describe pod -l app=mysql
   kubectl logs -l app=mysql
   ```
4. Fix issues like mismatched selectors, incorrect `DB_HOST`, or missing credentials in the ConfigMap.

## Conclusion
Troubleshooting Kubernetes applications requires a systematic approach, starting from service accessibility to pod and configuration checks. By verifying selectors, endpoints, logs, and environment variables, you can diagnose and resolve issues in a two-tier application like a web frontend and MySQL backend. Proper namespace management and corrective actions (e.g., updating ConfigMaps or reapplying YAML) ensure reliable application performance.
