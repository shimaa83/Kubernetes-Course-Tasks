# Task 1: Kubernete Course

## 1. Project Overview

In this task, I created a Kubernetes Deployment with 3 replicas. Each Pod contains:

- **Init Container:** Runs before the main containers and executes `echo done`.
- **Main Container:** Runs Nginx and serves an HTML page.
- **Sidecar Container:** Updates the HTML page with the current date and time every 5 seconds.

The project uses `shareProcessNamespace: true` to allow the Sidecar to access Nginx processes.

## 2. Environment Setup

Tools used:
- Docker Desktop
- WSL2 (Ubuntu)
- Minikube
- kubectl
- Calico CNI

I started a Minikube cluster with 3 nodes:

```bash
minikube start --nodes 3 --cni=calico --cpus 2 --memory 2200 --driver=docker
```

I checked the cluster nodes:

```bash
kubectl get nodes
```

## 3. Create Namespace and Deployment

I created a namespace called `webapp` and a Deployment called `web-deployment`.

The Deployment configuration includes:

- 3 replicas.
- Nginx container using `nginx:1.27`.
- Init Container using `busybox:1.36`.
- Sidecar Container using `busybox:1.36`.
- Shared process namespace.
- No volumes.

I saved the configuration in `deployment.yaml`.

## 4. Apply the Deployment

I applied the YAML file:

```bash
kubectl apply -f deployment.yaml
```

I checked the Deployment:

```bash
kubectl get deployments -n webapp
```

And checked the Pods:

```bash
kubectl get pods -n webapp
```

Expected result:

```text
READY   STATUS
2/2     Running
2/2     Running
2/2     Running
```

## 5. Test the Init Container

I checked the Init Container logs:

```bash
kubectl logs <pod-name> -n webapp -c init-done
```

Expected output:

```text
done
```

I also checked its status and exit code:

```bash
kubectl get pod <pod-name> -n webapp -o jsonpath='{.status.initContainerStatuses[*].state.terminated.reason}'
```

```bash
kubectl get pod <pod-name> -n webapp -o jsonpath='{.status.initContainerStatuses[*].state.terminated.exitCode}'
```

Expected results:

```text
Completed
0
```

## 6. Test Sidecar Container

I checked whether the Sidecar could see Nginx processes:

```bash
kubectl exec -n webapp <pod-name> -c time-updater -- pgrep nginx
```

The command returned Nginx process IDs, confirming that process namespace sharing was working.

## 7. Access the Nginx Web Page

I used port forwarding:

```bash
kubectl port-forward -n webapp deployment/web-deployment 8080:80
```

Then I opened:

```text
http://localhost:8080
```

The page displayed:

```text
hello from shimaa : YYYY-MM-DD HH:MM:SS
```

The Sidecar updates the page every 5 seconds.

## 8. Test Init Container Failure

To test what happens when the Init Container fails, I changed its command in `deployment.yaml` from:

```yaml
- echo done
```

To:

```yaml
- wrongcommand
```

Then I applied the changes:

```bash
kubectl apply -f deployment.yaml
```

I watched the Pods:

```bash
kubectl get pods -n webapp -w
```

The new Pod showed:

```text
Init:Error
Init:CrashLoopBackOff
```

This confirmed that when the Init Container fails, the main Nginx container and the Sidecar do not start in that Pod.

I checked the error logs:

```bash
kubectl logs <pod-name> -n webapp -c init-done
```

And inspected the Pod details:

```bash
kubectl describe pod <pod-name> -n webapp
```

## 9. Restore the Deployment

After testing the failure, I restored the original Init Container command:

```yaml
- echo done
```

Then I applied the configuration again:

```bash
kubectl apply -f deployment.yaml
```

I checked the rollout:

```bash
kubectl rollout status deployment/web-deployment -n webapp
```

Finally, I checked the Pods:

```bash
kubectl get pods -n webapp
```

Expected result: 3 Pods with `2/2 Running`.

## 10. Conclusion

In this task, I learned how to:

- Create a Kubernetes cluster with 3 nodes.
- Create a Namespace and Deployment.
- Use Init Containers and Sidecar Containers.
- Share processes between containers.
- Update files inside another container using `/proc`.
- Expose an application using port forwarding.
- Test Init Container failure and understand its effect on the Pod.

