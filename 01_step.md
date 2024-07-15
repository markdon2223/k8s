

### 1. Helm Dependency Update
Update Helm chart dependencies to ensure all required dependencies are installed and up-to-date.

```sh
helm dependency update
```

### 2. Initial Helm Installation and Upgrade
Install or upgrade the Helm release with specified values files.

```sh
helm upgrade -n cvat cvat1 -i --create-namespace ./helm-chart -f ./helm-chart/values.yaml -f ./helm-chart/values.override.yaml
```

### 3. Helm Upgrade
Upgrade the existing Helm release with specified values files.

```sh
helm upgrade -n cvat cvat1 ./helm-chart -f ./helm-chart/values.yaml -f ./helm-chart/values.override.yaml
```

### 4. Check Helm Release Status
Retrieve the status of the Helm release to ensure it is deployed correctly.

```sh
helm status -n cvat cvat1
```

### 5. Get Pods in Namespace
List all pods in the `cvat` namespace to check their status.

```sh
kubectl get pods -n cvat
```

### 6. Create Superuser
Create a superuser for CVAT by executing the Django `createsuperuser` command within the backend pod.

```sh
# Replace <backend-pod-name> with the actual pod name
BACKEND_POD_NAME=$(kubectl get pods -n cvat -l app.kubernetes.io/name=cvat-backend -o jsonpath="{.items[0].metadata.name}")
kubectl exec -it --namespace cvat $BACKEND_POD_NAME -c cvat-backend -- python manage.py createsuperuser
```

### 7. Get Events in Namespace
Retrieve events in the `cvat` namespace sorted by creation timestamp to debug issues.

```sh
kubectl get events -n cvat --sort-by='.metadata.creationTimestamp'
```

### 8. Check Persistent Volume Claims (PVC)
List all PVCs in the `cvat` namespace to check their status.

```sh
kubectl get pvc -n cvat
```

### 9. Ingress Status
Get the status of the Ingress resources in the `cvat` namespace.

```sh
kubectl get ingress -n cvat
```

### 10. Uninstall Helm Release
Uninstall the Helm release if it is no longer needed.

```sh
helm uninstall -n cvat cvat1
```

### 11. Get Services in Namespace
List all services in the `cvat` namespace to check their status and external IPs.

```sh
kubectl get svc -n cvat
```

### 12. Upgrade Only if Needed
Perform an upgrade only if you need to update the deployment.

```sh
helm upgrade -n cvat cvat1 ./helm-chart -f ./helm-chart/values.yaml -f ./helm-chart/values.override.yaml
```

### Example Workflow

Here’s a step-by-step example workflow to manage your CVAT deployment:

1. **Update Dependencies**:

    ```sh
    helm dependency update
    ```

2. **Install or Upgrade Deployment**:

    ```sh
    helm upgrade -n cvat cvat1 -i --create-namespace ./helm-chart -f ./helm-chart/values.yaml -f ./helm-chart/values.override.yaml
    ```

3. **Check Deployment Status**:

    ```sh
    helm status -n cvat cvat1
    ```

4. **Verify Pods**:

    ```sh
    kubectl get pods -n cvat
    ```

5. **Create Superuser**:

    ```sh
    BACKEND_POD_NAME=$(kubectl get pods -n cvat -l app.kubernetes.io/name=cvat-backend -o jsonpath="{.items[0].metadata.name}")
    kubectl exec -it --namespace cvat $BACKEND_POD_NAME -c cvat-backend -- python manage.py createsuperuser
    ```

6. **Check Events**:

    ```sh
    kubectl get events -n cvat --sort-by='.metadata.creationTimestamp'
    ```

7. **Check PVC Status**:

    ```sh
    kubectl get pvc -n cvat
    ```

8. **Verify Ingress**:

    ```sh
    kubectl get ingress -n cvat
    ```

9. **List Services**:

    ```sh
    kubectl get svc -n cvat
    ```

10. **Uninstall Deployment** (if needed):

    ```sh
    helm uninstall -n cvat cvat1
    ```

By following these documented steps, you can effectively manage your CVAT deployment on Kubernetes using Helm.
