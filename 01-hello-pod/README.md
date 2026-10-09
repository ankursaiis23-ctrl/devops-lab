\# Exercise 1: Hello Pod



\## Objective

Run a first application (nginx) inside Kubernetes using Minikube, expose it with a NodePort Service, and access it from the browser.



\## Environment

\- Windows 11, Docker Desktop

\- Minikube v1.39.0 (Docker driver)

\- kubectl v1.37.0



\## Steps and commands



1\. Start the cluster

```

minikube start --driver=docker

kubectl get nodes

```



2\. Create a Pod running nginx

```

kubectl run hello-k8s --image=nginx --port=80

```



3\. Verify the Pod is running

```

kubectl get pods

```

!\[Pod running](pods.png)



4\. Expose the Pod as a NodePort Service

```

kubectl expose pod hello-k8s --type=NodePort --port=80

kubectl get svc

minikube service hello-k8s --url

```

!\[Service](service.png)



5\. Open the URL in the browser

!\[nginx welcome page](nginx.png)



\## What I learned

\- A Pod is the smallest unit in Kubernetes and runs one or more containers.

\- `kubectl run` creates a single Pod. Nothing recreates it if it dies, so a Deployment is better for production.

\- A Service gives Pods a stable address. NodePort exposes the app on a port of the node (here 80:30459).

\- With the Docker driver on Windows, `minikube service --url` creates a tunnel, so the terminal must stay open.



\## Cleanup

```

kubectl delete svc hello-k8s

kubectl delete pod hello-k8s

```

