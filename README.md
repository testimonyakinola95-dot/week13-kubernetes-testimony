# Week 13 Kubernetes Practical – Nginx Deployment and Service Networking

## 1. Project Overview

This project demonstrates the deployment and management of a containerised Nginx web application using Kubernetes.

The practical was completed using a local Kubernetes cluster created with Kind (Kubernetes in Docker).

The project covers:

- Creating a Kubernetes namespace.
- Deploying an Nginx application.
- Configuring a ClusterIP Service.
- Configuring readiness probes and container resource limits.
- Scaling a Deployment from one replica to two replicas.
- Testing communication between Pods using a BusyBox client.
- Introducing and troubleshooting a Service selector failure.
- Restoring the Service and verifying application connectivity.

## 2. Tools and Technologies

- Kubernetes v1.35.0
- Kind v0.31.0
- Docker Desktop
- kubectl
- Nginx (`nginx:stable-alpine`)
- BusyBox (`busybox:1.37`)
- Ubuntu on WSL2

## 3. Project Structure

```text
week13-kubernetes/
├── deployment.yaml
├── service.yaml
├── README.md
└── evidence/
```

## 4. Kubernetes Configuration

### Namespace

A namespace called `lw-week13` was created to organise and isolate the resources used in this practical.

### Deployment

The Deployment is named `lw-web` and uses the `nginx:stable-alpine` container image.

The final configuration uses two replicas to maintain two running instances of the Nginx application.

Container configuration:

- Container port: 80
- Readiness probe: HTTP GET `/` on port 80
- Initial delay: 3 seconds
- Probe period: 5 seconds
- CPU request: 100m
- Memory request: 64Mi
- CPU limit: 250m
- Memory limit: 128Mi

### Service

A ClusterIP Service named `lw-web` exposes the Nginx Pods internally within the Kubernetes cluster.

- Service type: ClusterIP
- Service port: 80
- Target port: 80
- Protocol: TCP
- Selector: `app=lw-web`

The Service selector matches the labels assigned to the Nginx Pods, allowing traffic to reach the application.

## 5. Deployment and Verification

The Kubernetes resources were applied using:

```bash
kubectl apply -f deployment.yaml -f service.yaml
```

The Deployment rollout was checked using:

```bash
kubectl -n lw-week13 rollout status deployment/lw-web
```

Resources were inspected using:

```bash
kubectl -n lw-week13 get deployment,service,pods
```

The application was also accessed locally using port forwarding:

```bash
kubectl -n lw-week13 port-forward service/lw-web 8080:80
```

The Nginx response was tested using:

```bash
curl -I http://localhost:8080
```

The HTTP test returned `200 OK`, confirming that the Nginx application was accessible.

## 6. Scaling the Deployment

The Deployment was initially configured with one replica.

It was then scaled to two replicas using:

```bash
kubectl -n lw-week13 scale deployment/lw-web --replicas=2
```

The saved `deployment.yaml` file was updated to use:

```yaml
replicas: 2
```

The updated configuration was applied using:

```bash
kubectl apply -f deployment.yaml
```

The final Deployment showed two available replicas, with both Nginx Pods in the Running and Ready state.

## 7. Internal Service Communication Test

A temporary BusyBox client Pod was created:

```bash
kubectl -n lw-week13 run lw-client --image=busybox:1.37 --restart=Never --command -- sleep 3600
```

The client Pod was checked for readiness:

```bash
kubectl -n lw-week13 wait --for=condition=Ready pod/lw-client --timeout=120s
```

Communication with the Nginx Service was tested using:

```bash
kubectl -n lw-week13 exec lw-client -- wget -T 5 -qO- http://lw-web
```

The request successfully returned the Nginx welcome page.

This demonstrated that the BusyBox Pod could resolve the Service name and communicate with the Nginx application through the Kubernetes Service.

## 8. Fault Injection and Troubleshooting

To demonstrate Kubernetes Service troubleshooting, the Service selector was deliberately changed from `app=lw-web` to `app=wrong-label`.

The incorrect selector was applied using:

```bash
kubectl -n lw-week13 patch service lw-web --type=merge -p '{"spec":{"selector":{"app":"wrong-label"}}}'
```

The internal connectivity test was repeated:

```bash
kubectl -n lw-week13 exec lw-client -- wget -T 5 -qO- http://lw-web
```

The request failed with a connection-refused error.

### Diagnosis

The following commands were used to investigate the issue:

```bash
kubectl -n lw-week13 get pods --show-labels
```

```bash
kubectl -n lw-week13 describe service lw-web
```

```bash
kubectl -n lw-week13 get endpointslices -l kubernetes.io/service-name=lw-web -o yaml
```

The investigation showed that:

- The Nginx Pods were still Running and Ready.
- The Pods had the label `app=lw-web`.
- The Service selector had been changed to `app=wrong-label`.
- The Service had no matching backend endpoints.
- The EndpointSlice contained no backend addresses.

### Root Cause

The Service selector did not match the labels assigned to the Nginx Pods.

Because Kubernetes Services use selectors to identify their backend Pods, the incorrect selector prevented the Service from discovering the Nginx Pods.

### Resolution

The correct selector was restored in `service.yaml`:

```yaml
selector:
  app: lw-web
```

The corrected Service configuration was applied using:

```bash
kubectl apply -f service.yaml
```

The internal connectivity test was repeated and successfully returned the Nginx welcome page.

The Service recovered two backend endpoints, and the Deployment remained at two available replicas.

## 9. Final Verification

The following checks were used to verify the final state:

```bash
kubectl -n lw-week13 get deployment lw-web
```

```bash
kubectl -n lw-week13 get pods -l app=lw-web
```

```bash
kubectl -n lw-week13 describe service lw-web
```

```bash
kubectl -n lw-week13 get endpointslices -l kubernetes.io/service-name=lw-web
```

```bash
kubectl -n lw-week13 exec lw-client -- wget -T 5 -qO- http://lw-web
```

Final results:

- Deployment: 2/2 replicas available.
- Nginx Pods: two Running and Ready Pods.
- Service: ClusterIP.
- Service selector: `app=lw-web`.
- Service endpoints: two backend IP addresses.
- Internal HTTP communication: successful.

## 10. Cleanup

The Kubernetes resources created for this practical can be removed using:

```bash
kubectl delete namespace lw-week13
```

To delete the local Kind cluster:

```bash
kind delete cluster --name lw-week13
```

**Note:** These commands are for cleanup after assessment and should not be run while the practical environment is still needed.

## 11. Conclusion

This practical provided hands-on experience with Kubernetes application deployment, Service networking, scaling, readiness probes, resource management, and troubleshooting.

It demonstrated how Kubernetes Services use label selectors to discover backend Pods and how an incorrect selector can interrupt application connectivity even when the Pods themselves remain healthy.

The issue was diagnosed through Pod labels, Service inspection, and EndpointSlice inspection. Restoring the correct selector successfully restored communication between the BusyBox client and the Nginx application.
