Student: **Joanne Mesadieu**  
Course: **Operating Systems**  
Instructor: **Dr. Gupta**  
Date: **September 22, 2026**

> Preparation draft: run the assignment, replace all bracketed fields, and add genuine screenshots before submitting. No runtime results or screenshots have been supplied yet.

## Application and concepts

This project follows the assignment's supplied official Nginx configuration. The assignment also refers to the web app from Assignment 1; confirm whether that app must replace the Nginx welcome page before submitting. The supplied files alone serve the default Nginx page, not a previous custom app.

Kubernetes manages containerized applications by keeping their running state aligned with a declared desired state. A Pod is the smallest deployable unit and contains one or more containers that share networking and can share storage. A Deployment manages replicas of a Pod template through ReplicaSets and supports rolling updates. A Service provides a stable network endpoint for matching Pods, even when individual Pods are replaced.

## Project structure

```text
firstname_lastname_kubernetes_assignment/
├── README.md
├── deployment.yaml
├── service.yaml
└── screenshots/
```

Rename the folder using your own first and last name. Name the public GitHub repository `firstname-lastname-kubernetes-assignment`.

## Installation

Install Docker Desktop, Git, and kubectl. In Docker Desktop settings, enable Kubernetes and wait for it to start. Minikube or Kind is also acceptable, but only one local cluster is needed.

Official setup instructions:

- [Docker Desktop Kubernetes](https://docs.docker.com/desktop/features/kubernetes/)
- [Install kubectl on Windows](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)
- [Install Git](https://git-scm.com/downloads)

Open PowerShell in this project folder. Verify the tools and confirm that the selected context belongs to your local assignment cluster:

```powershell
docker --version
git --version
kubectl version --client
kubectl config current-context
kubectl get nodes
```

Wait for the node to show `Ready`. The commands below use the current context and namespace; use the same context and namespace throughout.

## 1. Deploy two replicas

```powershell
kubectl apply -f deployment.yaml
kubectl rollout status deployment/student-web-deployment --timeout=180s
kubectl get deployments
kubectl get pods -l app=student-web
kubectl get pods -l app=student-web -o wide
```

Capture two Pods with `STATUS` equal to `Running` and `READY` equal to `1/1`. Save the screenshot as `screenshots/01-two-pods.png`.

## 2. Create the Service and open the app

```powershell
kubectl apply -f service.yaml
kubectl get services
kubectl port-forward service/student-web-service 8080:80
```

Keep this terminal open and visit http://localhost:8080 in your browser. Save the Nginx welcome page screenshot as `screenshots/02-nginx-page.png`. Use a second PowerShell terminal in the project folder for subsequent commands.

For Minikube, `minikube service student-web-service` is an alternative. Although this Service has type `NodePort`, port forwarding uses a local tunnel and does not require browsing to the allocated NodePort.

## 3. Scale up and down

```powershell
kubectl scale deployment student-web-deployment --replicas=4
kubectl rollout status deployment/student-web-deployment --timeout=180s
kubectl get pods -l app=student-web
```

Save `screenshots/03-four-pods.png` showing four running, ready Pods.

```powershell
kubectl scale deployment student-web-deployment --replicas=2
kubectl rollout status deployment/student-web-deployment --timeout=180s
kubectl get pods -l app=student-web
```

Wait until only two Pods remain and neither is terminating before proceeding.

## 4. Delete a Pod and observe recovery

These PowerShell commands select one currently running application Pod and delete it by name:

```powershell
kubectl get pods -l app=student-web
$deletedPod = kubectl get pods -l app=student-web --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}'
Write-Output "Deleting Pod: $deletedPod"
kubectl delete pod $deletedPod
kubectl get pods -l app=student-web
```

Save the command and deletion output as `screenshots/04-pod-deletion.png`. Then run:

```powershell
kubectl get pods -l app=student-web -w
```

When the replacement is running and ready, press Ctrl+C and run:

```powershell
kubectl get pods -l app=student-web -o wide
```

Save `screenshots/05-replacement-pod.png`, showing the new Pod name and two ready Pods. Replacement may be fast enough that the first listing already shows the new Pod.

Expected explanation (confirm against your run): The Deployment's ReplicaSet maintains a desired count of two Pods. Deleting a managed Pod causes the controller to create a replacement with a new name and UID so the application returns to two replicas. This is a replacement Pod, rather than a restart of the deleted Pod.

Actual observation: **[Record the deleted Pod name, replacement Pod name, and observed result.]**

Port forwarding selects one Pod; if that Pod is deleted, the forwarding session ends. Restart `kubectl port-forward service/student-web-service 8080:80` if needed. Multiple replicas do not make an individual port-forward session fail over automatically.

## 5. Inspect resources, events, and logs

```powershell
$inspectPod = kubectl get pods -l app=student-web --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}'
kubectl describe pod $inspectPod
kubectl logs $inspectPod -c nginx-container
kubectl get pods -l app=student-web -o wide
kubectl get deployment student-web-deployment
kubectl get events --sort-by=.metadata.creationTimestamp
```

Refresh the browser page to generate web requests if desired. A request may reach a different replica, so not every Pod will necessarily show its access log.

Fill this table using actual output before the image update:

PS C:\Users\joann> kubectl describe deployment student-web-deployment
Name:                   student-web-deployment
Namespace:              default
CreationTimestamp:      Tue, 22 Sep 2026 14:15:17 -0400
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=student-web
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=student-web
  Containers:
   nginx-container:
    Image:         nginx:stable-alpine
    Port:          80/TCP
    Host Port:     0/TCP
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  student-web-deployment-76b6d487f6 (0/0 replicas created)
NewReplicaSet:   student-web-deployment-864884c498 (2/2 replicas created)

## 6. Update the image

Use this single-line command in PowerShell:

```powershell
kubectl set image deployment/student-web-deployment nginx-container=nginx:stable-alpine
kubectl rollout status deployment/student-web-deployment --timeout=180s
kubectl rollout history deployment/student-web-deployment
kubectl describe deployment student-web-deployment
kubectl get pods -l app=student-web -o wide
```

Save `screenshots/06-updated-image.png`, showing successful rollout output and `Image: nginx:stable-alpine`. More than two Pods may exist temporarily during the rolling update.

The deployment file preserves the initial `nginx:alpine` image required by the assignment. The update command changes the live Deployment; applying the original file again would restore its original image.

## 7. Clean up

Stop the port-forward terminal with Ctrl+C, then run:

```powershell
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
kubectl get deployments
kubectl get pods
kubectl get services
```

Verify that `student-web-deployment`, its Pods, and `student-web-service` are gone. Leave unrelated resources and the default `kubernetes` Service in place. Optionally save a cleanup screenshot as `screenshots/07-cleanup.png`.

## Screenshot evidence

After saving real screenshots with these names, uncomment the corresponding Markdown image lines below to display them on GitHub.

https://github.com/jomesadieu/Joanne_Mesadieu_kubernetes_assignment/blob/f03d7b4bab569a54bfbf7e1a8761d55227a684f9/01-two-pods.png

https://github.com/jomesadieu/Joanne_Mesadieu_kubernetes_assignment/blob/f03d7b4bab569a54bfbf7e1a8761d55227a684f9/02-nginx-page.png

https://github.com/jomesadieu/Joanne_Mesadieu_kubernetes_assignment/blob/f03d7b4bab569a54bfbf7e1a8761d55227a684f9/03-four-pods.png

https://github.com/jomesadieu/Joanne_Mesadieu_kubernetes_assignment/blob/f03d7b4bab569a54bfbf7e1a8761d55227a684f9/04-pod-deletion.png

https://github.com/jomesadieu/Joanne_Mesadieu_kubernetes_assignment/blob/f03d7b4bab569a54bfbf7e1a8761d55227a684f9/05-replacement-pod.png

https://github.com/jomesadieu/Joanne_Mesadieu_kubernetes_assignment/blob/f03d7b4bab569a54bfbf7e1a8761d55227a684f9/06-updated-image.png

https://github.com/jomesadieu/Joanne_Mesadieu_kubernetes_assignment/blob/c83fa40aa1ae080b5708a6828b75dc0ad5755680/07-cleanup.png.png

## Problems encountered and resolutions

**[Record only problems you actually encountered and how you resolved them. If none occurred, state that after completing the run.]**

Troubleshooting reference, not a record of completed work:

- If kubectl cannot connect, check that the local cluster is running and the current context points to it.
- For `Pending` or `ImagePullBackOff`, inspect `kubectl describe pod` and its events for resource, image-download, or network errors.
- If port 8080 is occupied, use `kubectl port-forward service/student-web-service 8081:80` and browse to http://localhost:8081.
- If the forwarding session ends after deletion or rollout, start it again.

## Questions

### 1. What is Kubernetes?

Kubernetes is an open-source platform for managing containerized applications. It automates tasks such as deployment, scaling, networking, and replacing failed workloads to maintain the desired application state.

### 2. What is the difference between a Docker container and a Kubernetes Pod?

A Docker container is an isolated running instance of a container image. A Kubernetes Pod wraps one or more containers that are scheduled together and share a network namespace and can share storage volumes. Kubernetes manages these containers through Pods.

### 3. What is the purpose of a Deployment?

A Deployment declares the desired application configuration, including its container image and replica count. It manages ReplicaSets to maintain that configuration and supports controlled rolling updates and rollbacks.

### 4. What is the purpose of a Service?

A Service provides a stable network address for a set of Pods selected by labels. It directs traffic to eligible backends so clients do not have to track changing Pod IP addresses.

### 5. Why did Kubernetes recreate the deleted Pod?

The Deployment's ReplicaSet was configured to maintain two replicas. Once a managed Pod was deleted, its controller created a new Pod to restore the desired count.

### 6. What is meant by scaling an application?

Scaling means adjusting an application's capacity to handle work. In this assignment, horizontal scaling increases the replica count from two to four Pods and then reduces it back to two.

### 7. What is the difference between kubectl apply and kubectl delete?

`kubectl apply` creates or updates resources using configuration files. `kubectl delete` removes the specified resources from the cluster; it does not delete the local YAML files.

### 8. Why are multiple replicas useful?

Multiple replicas allow requests to be served by more than one application instance and can help maintain availability when one instance fails or is updated. They can also increase capacity when traffic is distributed among them. Replicas on a single local node do not protect against failure of that entire node.

## Submission checklist

- [ ] Replace student and course placeholders and rename the project folder.
- [ ] Resolve whether the instructor requires the Assignment 1 web app.
- [ ] Run every assignment step and fill in actual observations.
- [ ] Add six required screenshots and enable their image links above.
- [ ] Record actual problems and resolutions.
- [ ] Remove unfinished placeholders and this preparation notice.
- [ ] Create a public repository named `firstname-lastname-kubernetes-assignment`.
- [ ] Upload README.md, deployment.yaml, service.yaml, and screenshots/ to the repository root.
- [ ] Check the public repository and images while signed out.
- [ ] Submit only the public repository URL.

## References

- [Kubernetes concepts](https://kubernetes.io/docs/concepts/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [kubectl command reference](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)
- [Port-forward behavior](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/)
