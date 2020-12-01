# Cheat sheet

```
$ cd /Users/preetam/Projects/k3d-environment
```

---

```
$ docker volume create local_registry
```

---

```
$ docker container run -d \
  --name registry.localhost \
  -v local_registry:/var/lib/registry \
  --restart always \
  -p 5000:5000 \
  registry:2
```

---

```
$ cat /etc/hosts | grep registry.localhost
```

---

```
$ docker pull tomcat:latest
$ docker tag tomcat:latest registry.localhost:5000/tomcat:latest
$ docker push registry.localhost:5000/tomcat:latest
$ docker image rm tomcat:latest
$ docker image rm registry.localhost:5000/tomcat:latest
$ docker pull registry.localhost:5000/tomcat:latest
```

---

```
$ k3d cluster create preet-cluster \
  --k3s-server-arg --flannel-backend="vxlan" \
  --k3s-server-arg --disable=traefik \
  --servers 1 \
  --agents 2 \
  --volume /Users/preetam/Projects/k3d-environment/k3d-registries.yaml:/etc/rancher/k3s/registries.yaml

```

---

```
$ docker network list | grep preet-cluster
$ docker network connect b5bbcffecdaa registry.localhost
$ docker network inspect k3d-preet-cluster
```

---

(After docker restart)
```
$ k3d cluster start preet-cluster
```
