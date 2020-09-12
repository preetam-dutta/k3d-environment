# K3D Development Environment Setup

This is a PoC that demonstrates Kubernetes development environment setup on K3D. 
This has been tested on Win10 with WSL and MacOS. It is supposed to work on Linux distributions.

K3D is a lightweight Kubernetes distribution based on K3S. For details check references section.


# Pre-requisites
- The pre-requisite is to have *Docker For Desktop* already setup either for Win10 (with Linux containers) or MacOS. 
- WSL for Win10 (to install WSL on Win10 follow - https://nickjanetakis.com/blog/setting-up-docker-for-windows-and-wsl-to-work-flawlessly)

# K3D installation

## MacOS
For MacOS (and Linux distro) use https://brew.sh/

```
brew install k3d
```

## Win10 (with WSL)
Execute the below command on the WSL terminal

```
curl -s https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash
```

## Test K3D installation
Execute **k3d --version** to ensure installation is successfull

```
>> k3d --version

k3d version v3.0.1
k3s version v1.18.6-k3s1 (default)
```

# Start Kubernetes cluster

## Example of Kubernetes cluster with default options

```
>> k3d cluster create <cluster-name>
```

## Example of Kubernetes cluster with 1 master and 2 worker nodes

```
>> k3d cluster create <cluster-name> \
  --servers 1 \
  --agents 2
```

## Example of Kuberneter with K3S server args

```
>> k3d cluster create <cluster-name> \
  --k3s-server-arg --flannel-backend="vxlan" \
  --k3s-server-arg --disable=traefik \
  --verbose \
  --servers 1 \
  --agents 2
```

## Example of Kubernetes with custom Docker registries
For using this example you would first need to setup the **registries.yaml** file, refer the section **Docker repository setup** on how to do it first. 

```
>> k3d cluster create <cluster-name> \
  --k3s-server-arg --flannel-backend="vxlan" \
  --k3s-server-arg --disable=traefik \
  --volume $(pwd)/k3d-registries.yaml:/etc/rancher/k3s/registries.yaml \
  --verbose \
  --servers 1 \
  --agents 2
```

## Other cluster create options
For further details refer command details - **k3d cluster create --help**

## Test cluster

Check cluster-info

```
>> kubectl cluster-info

Kubernetes master is running at https://0.0.0.0:59117
CoreDNS is running at https://0.0.0.0:59117/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:59117/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'
```

Check nodes

```
>> kubectl get nodes

NAME                           STATUS   ROLES    AGE     VERSION
k3d-preet-cluster-3-agent-1    Ready    <none>   7h28m   v1.18.6+k3s1
k3d-preet-cluster-3-agent-0    Ready    <none>   7h28m   v1.18.6+k3s1
k3d-preet-cluster-3-server-0   Ready    master   7h28m   v1.18.6+k3s1
```


# Docker local repository setup
Setup local Docker repository for the newly created K3D cluster.

## Setup local registry container

Create **Docker Volume** for the repository

```
docker volume create local_registry
```

Run the **Repository Container**

```
>> docker container run -d \
  --name registry.localhost \
  -v local_registry:/var/lib/registry \
  --restart always \
  -p 5000:5000 \
  registry:2
```

Update /etc/host to point registries.yaml to 127.0.0.0, so that you can push images.

```
>> cat /etc/hosts | grep registry.localhost

127.0.0.1       registry.localhost

```

## Test Repositoy Container is working properly

Pull, tag and push an image to the reposit

```
>> docker pull nginx:latest
>> docker tag nginx:latest registry.localhost:5000/nginx:latest
>> docker push registry.localhost:5000/nginx:latest
```

Delete the cached image

```
>> docker image rm nginx:latest
>> docker image rm registry.localhost:5000/nginx:latest
```

Once the local cache is deleted, pull the image from the local repo to ensure the Repository Container is working properly

```
>> docker pull registry.localhost:5000/nginx:latest
```

## Connect the local repository(or any private repo) to the Kubernetes cluster created by K3D

### Create k3d-registries.yaml

**NOTE** Ensure Docker For Desktop have access to the path where k3d-registries.yaml is located

On present working directory(or any other directory) create k3d-registries.yaml

```
>> cat k3d-registries.yaml

mirrors:
  "registry.localhost":
    endpoint:
      - http://registry.localhost:5000
```

or, if you also want to connect to a private repo then -

```
>> cat k3d-registries.yaml

mirrors:
  "registry.localhost":
    endpoint:
      - http://registry.localhost:5000
  "your.private.repo.fqdn":
    endpoint:
      - https://your.private.repo.fqdn

configs:
  your.private.repo.fqdn:
    auth:
      username: <user>
      password: <pass>
```

### Start K3D cluster with k3d-registries.yaml

Delete the cluster you have already created in the previous section (**optional** only if you have already create the cluster without k3d-registries.yaml setup)

```
>> k3d cluster delete <cluster-name>
```

Execute the below command from the path where you have create the **k3d-registries.yaml** file

```
>> k3d cluster create <cluster-name> \
  --k3s-server-arg --disable=traefik \
  --volume $(pwd)/k3d-registries.yaml:/etc/rancher/k3s/registries.yaml \
  --verbose \
  --servers 1 \
  --agents 2
```

### Bridge Registry Container and K3D cluster

Get the cluster network

```
>> docker network list | grep <cluster-name>
```

Bridge the network

```
>> docker network connect <docker-network from previous command> registry.localhost

e.g. docker network connect k3d-<cluster-name> registry.localhost
```

Verify the registry.localhost is listed in the k3d cluster's network

```
>> docker network inspect k3d-<cluster-name>
```

### Test if a Pod is able to fetch image from the local regisry

Create a simple pod, that will use the image from the local repo

```
>> cat pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: simple-pod
spec:
  containers:
    - image: registry.localhost/nginx:latest
      name: simple-pod
      imagePullPolicy: IfNotPresent
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "100m"
          memory: "128Mi"
```

Test by running the pod, if all ok the pod will comeup

```
kubectl apply -f pod.yaml
```

# References
- https://github.com/rancher/k3d
- https://k3d.io/usage/guides/registries/
- https://brew.sh/