# "Remote" Docker Desktop Kubernetes
## Description
A simple work-around to make Docker Desktop Kubernetes remotely accessible over Local Area Network. This is useful if you want to dedicate a machine for a single-node local development cluster without complications of setting up a standard K8s cluster.

## What's the core idea?
As of 8 August 2024, Docker Desktop Kubernetes while being a fantastic easy-to-use local kubernetes cluster, it is only listening locally on kubernetes.docker.internal or specifically 127.0.0.1. 
In order to be able to access the control plane, you need to expose 6443. One way to do it is to use `socat`. This idea was obtained from a blog by here @evgeniyzaydun -> https://medium.com/@evgeniyzaydun/connecting-to-docker-desktop-kubernetes-remotely-a-guide-for-development-purposes-11cd2bc5c474
However @evgeniyzaydun post-socat steps can be quite overwhelming so I want to present an easier but less secure way. This is to be used for home lab, not for production. And you shouldn't be using Docker Desktop Kubernetes for production anyway! :D 

## Pre-requisites
This article assumes that you have already:
1. Installed Docker Desktop on your Windows operating system
2. Enabled Kubernetes

## Steps
Terminology:
- Node machine (i.e. where you want your Docker Desktop Kubernetes to be installed on and deployed)
- User machine (i.e. where you will be remotely accessing from)
1. Run the following docker command:
```
docker run -d --name socat-remote-docker-desktop -p 7443:7443 alpine/socat TCP-LISTEN:7443,fork TCP:kubernetes.docker.internal:6443
```
Docker Desktop Kubernetes listens on your localhost 127.0.0.1:6443 so you will need to use another port. 

2. Retrieve the `.kube config`, make some modifications and before updating your `User machine`'s `.kube config`:

Original:
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tL...SNIPPED...mUKNkNTdVE5UWRDR1BzCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://127.0.0.1:6443
  name: docker-desktop
contexts:
- context:
    cluster: docker-desktop
    user: docker-desktop
  name: docker-desktop
current-context: docker-desktop
kind: Config
preferences: {}
users:
- name: docker-desktop
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURJekNDQWd1Z0F3SUJBZ0lJQzRzcVlDNGVpM3...SNIPPED...Y0F2UWNSOTc3YQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFU...SNIPPEDQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
```
You are expected to:
- Delete the `certificate-authority-data`
- Add the `insecure-skip-tls-verify: true` field
- For each section (i.e. clusters, contexts and users), copy and paste them to your `User machine`'s .kube config if it's mixed in with other kubernete clusters configuration. If you do not have any existing kubernetes configurations, please feel free to create the `.kube config`. Please note that this `.kube config` file is automatically created when you enable Kubernetes on Docker Desktop. However, since your `User machine` is not the one installed with Docker Desktop Kubernetes, it might not have it. Anyway, the file is located found under the following path: `C:\users\<your_username>\.kube\config`
- (Optional) Name change from `docker-desktop` to your preferred e.g. `remote-docker-desktop`. Please note that you have to reflect the change across the various fields. 

Modified:
```
apiVersion: v1
clusters:
- name: remote-docker-desktop
    cluster:
      server: https://192.168.1.18:7443
      insecure-skip-tls-verify: true
contexts:
- context:
    cluster: remote-docker-desktop
    user: remote-docker-desktop
  name: remote-docker-desktop
current-context: remote-docker-desktop
kind: Config
preferences: {}
users:
- name: remote-docker-desktop
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURJekNDQWd1Z0F3SUJBZ0lJQzRzcVlDNGVpM3...SNIPPED...Y0F2UWNSOTc3YQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFU...SNIPPEDQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
```

3. To ensure it works, you may use the following command:
```
kubectl get nodes
```
You should observe a response similar to the following:
```
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   60m   v1.30.2
```

## Exposing external IP address
If you do a `kubectl get svc -A`, what is suppose to be an external IP address was assigned to a `localhost` as so:
```
k get svc -A
NAMESPACE     NAME                                    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                 AGE
myapp         internet-facing-svc                     LoadBalancer   10.102.108.247   localhost     80:31564/TCP,443:31973/TCP              71s
idp           identity-provider-svc                   LoadBalancer   10.109.128.17    localhost     8080:30959/TCP,8443:30297/TCP           70s
...
...
```
For each external IP address, you may use `socat` again to expose them on your LAN. 
For example if you wish to expose a 443 service, you may on the `Node machine` run the following command:
```
docker run -d --name socat-web-service -p 443:443 alpine/socat TCP-LISTEN:443,fork TCP:kubernetes.docker.internal:443
```
