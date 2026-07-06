

Pure commands run straight in the terminal. For anything file based (Containerfile, manifest) the steps are always the same three: open the file with `vi`, type in the full content shown, then run the command.

> How to use `vi` quickly: type `vi name`, press `i` to start typing, type the content, press `Esc`, then type `:wq` and Enter to save and quit.

---

## 0. One-time setup

Questions 1 to 18 use Podman (already installed on the exam VM). Questions 19 to 43 use Kubernetes, so start minikube first when you reach them:

```bash
minikube start
kubectl get nodes
```

Reusable diagnosis commands:

```bash
podman logs <name>                 # why a container exited
kubectl describe <kind> <name>     # events at the bottom explain most failures
kubectl get events --sort-by=.lastTimestamp
```

---

## 1. podman run -d --name web -p 80:8080 docker.io/library/nginx (nginx listens on 80; the site isn't reachable on the host port you expected — what's reversed?)

Mistake: `-p` is `HOST:CONTAINER`, so this maps host port 80 to container port 8080, but nginx listens on 80 inside the container.

Fix:

```bash
podman run -d --name web -p 8080:80 docker.io/library/nginx
curl http://localhost:8080
```

The container port must be 80 because that is where nginx listens. The host port on the left can be anything you like.

---

## 2. podman run -d --name db -e MYSQL_ROOT_PASSWORD docker.io/library/mysql (the container exits during init — inspect podman logs.)

Mistake: `-e MYSQL_ROOT_PASSWORD` passes the variable name with no value, so MySQL has no root password strategy and aborts initialization.

Fix:

```bash
podman run -d --name db -e MYSQL_ROOT_PASSWORD=secret docker.io/library/mysql
podman logs db
```

MySQL requires one of `MYSQL_ROOT_PASSWORD`, `MYSQL_ALLOW_EMPTY_PASSWORD`, or `MYSQL_RANDOM_ROOT_PASSWORD` with an actual value, otherwise init fails.

---

## 3. podman run -d --name app --network host -p 8080:80 docker.io/library/nginx (why is the -p flag effectively ignored here?)

Mistake: with `--network host` the container shares the host network stack directly, so there is no NAT layer and `-p` does nothing.

Fix, either drop host networking so `-p` works:

```bash
podman run -d --name app -p 8080:80 docker.io/library/nginx
```

or keep host networking and reach nginx directly on port 80:

```bash
podman run -d --name app --network host docker.io/library/nginx
curl http://localhost:80
```

Port publishing only applies to bridge networking. In host mode the container already uses the host's ports, so mapping is meaningless.

---

## 4. podman run -d --name c1 docker.io/library/busybox (the container goes straight to Exited (0) — why, and how do you keep it running?)

Mistake: busybox has no long running default command, so its main process finishes immediately and the container exits.

Fix:

```bash
podman run -d --name c1 docker.io/library/busybox sleep 1d
podman ps
```

A container lives only as long as its main process. Giving it a long running command like `sleep 1d` keeps it up.

---

## 5. podman run --rm -d --name job docker.io/library/alpine echo hello (then podman logs job fails — explain the --rm + detached gotcha.)

Mistake: the container runs `echo hello`, exits instantly, and `--rm` deletes it right away, so there is nothing left for `podman logs` to read.

Fix, drop `--rm` so the container and its logs survive:

```bash
podman run -d --name job docker.io/library/alpine echo hello
podman logs job
```

`--rm` auto removes the container the moment it exits, taking its logs with it. Use it for throwaway runs, not when you need to inspect output afterward.

---

## 6. podman run -d -p 8080:80 --memory 8m docker.io/library/mysql (the database never becomes healthy — what limit is the problem?)

Mistake: `--memory 8m` gives MySQL only 8 MB of RAM, far below what it needs, so it is killed or never finishes starting. It is also missing a root password.

Fix:

```bash
podman run -d -p 8080:80 --memory 512m -e MYSQL_ROOT_PASSWORD=secret docker.io/library/mysql
podman logs -f <name>
```

The memory limit was starving the database. Raising it to something realistic lets it start and pass its health check.

---

## 7. podman run -d --name a docker.io/library/alpine sleep 1d / podman run -d --name b docker.io/library/alpine sleep 1d / podman exec a ping b (name resolution fails — why, on the default network, and how do you fix it?)

Mistake: both containers run on the default network, which has no internal DNS, so the name `b` does not resolve.

Fix, put them on a user defined network:

```bash
podman network create mynet
podman rm -f a b
podman run -d --network mynet --name a docker.io/library/alpine sleep 1d
podman run -d --network mynet --name b docker.io/library/alpine sleep 1d
podman exec a ping -c1 b
```

Only user defined networks provide name based resolution. The default network does not, so containers there cannot find each other by name.

---

## 8. podman run -d --name web -v ./html:/usr/share/nginx/html docker.io/library/nginx (the files aren't visible in the container, or SELinux denies access — what mount flag is missing?)

Mistake: on an SELinux system the bind mount needs a relabel flag, without it the container context is denied access to the host files.

Fix:

```bash
podman run -d --name web -v ./html:/usr/share/nginx/html:Z docker.io/library/nginx
```

The `:Z` flag relabels the host directory for this container's private use (`:z` for a directory shared between containers). Without it SELinux blocks the container from reading the mounted files.

---

## 9. For each Containerfile/Dockerfile: identify the mistake, fix it, and explain what was wrong.

This is the heading for questions 10 to 18 below. Each shows a broken Containerfile, the fix, and why.

---

## 10. FROM debian:12 / RUN apt-get install -y nginx (the build fails to find the package — what's missing before install?)

Mistake: there is no package index in the base image until you refresh it, so `apt-get install` cannot find nginx.

Fix. Create the file:

```bash
vi Containerfile
```

Put this inside:

```dockerfile
FROM debian:12
RUN apt-get update && apt-get install -y nginx
```

Then build:

```bash
podman build -t fixed10 .
```

You must run `apt-get update` before `apt-get install` so the package lists exist. Combine them in one `RUN` so the update always matches the install.

---

## 11. FROM python:3.11 / COPY app.py /app/app.py / WORKDIR /app / CMD python app.py (the app starts but doesn't handle signals / can't be stopped cleanly — shell vs exec form?)

Mistake: `CMD python app.py` is shell form, so the process runs under `/bin/sh -c` and is not PID 1, which means it never receives stop signals cleanly.

Fix. Create the file:

```bash
vi Containerfile
```

Put this inside:

```dockerfile
FROM python:3.11
COPY app.py /app/app.py
WORKDIR /app
CMD ["python", "app.py"]
```

Then build:

```bash
podman build -t fixed11 .
```

The exec form (`CMD ["python","app.py"]`) makes python PID 1, so it receives `SIGTERM` and stops cleanly. Shell form wraps it in a shell that swallows the signal.

---

## 12. FROM node:20 / COPY . /app / WORKDIR /app / RUN npm install (every source change triggers a full npm install — reorder for layer caching.)

Mistake: copying all source before `npm install` means any code change invalidates the cache and reinstalls every dependency.

Fix. Create the file:

```bash
vi Containerfile
```

Put this inside:

```dockerfile
FROM node:20
WORKDIR /app
COPY package*.json /app/
RUN npm install
COPY . /app
```

Then build:

```bash
podman build -t fixed12 .
```

Copy only the dependency manifest first and install, then copy the rest of the source. The `npm install` layer is now cached and only reruns when `package.json` changes.

---

## 13. FROM alpine:3.20 / ENV PATH=/app/bin / RUN apk add --no-cache curl (after this, curl/sh aren't found — what did setting PATH break?)

Mistake: `ENV PATH=/app/bin` overwrites the whole PATH, so `/bin` and `/usr/bin` disappear and even `apk` and `sh` cannot be found.

Fix. Create the file:

```bash
vi Containerfile
```

Put this inside:

```dockerfile
FROM alpine:3.20
ENV PATH=/app/bin:$PATH
RUN apk add --no-cache curl
```

Then build:

```bash
podman build -t fixed13 .
```

Always append to PATH with `:$PATH` instead of replacing it, so the system directories stay on the path.

---

## 14. FROM ubuntu:24.04 / EXPOSE 8080 / CMD ["python3", "-m", "http.server", "3000"]

## 15. (you map -p 8080:8080 but nothing answers — what mismatch is there, and what does EXPOSE actually do?)

Mistake: the server listens on 3000 but you EXPOSE and map 8080, so nothing answers on the port you published. Also `EXPOSE` is only documentation, it does not publish anything. (Ubuntu also has no python3 preinstalled, so install it.)

Fix. Create the file:

```bash
vi Containerfile
```

Put this inside:

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y python3
EXPOSE 8080
CMD ["python3", "-m", "http.server", "8080"]
```

Then build and run with matching ports:

```bash
podman build -t fixed15 .
podman run -d -p 8080:8080 fixed15
curl http://localhost:8080
```

The listening port and the mapped container port must match, here both 8080. `EXPOSE` only records intent as metadata, the actual publishing is done by `-p` at run time.

---

## 16. FROM debian:12 / RUN apt-get update && apt-get install -y build-essential (the image is huge — how do you cut the size in the same layer?)

Mistake: the apt package lists stay in the image, bloating it, because they are not cleaned in the same layer that created them.

Fix. Create the file:

```bash
vi Containerfile
```

Put this inside:

```dockerfile
FROM debian:12
RUN apt-get update && apt-get install -y build-essential && rm -rf /var/lib/apt/lists/*
```

Then build:

```bash
podman build -t fixed16 .
```

The cleanup must be in the same `RUN` as the install. If you delete the lists in a later layer, the earlier layer still carries them and the image stays large.

---

## 17. FROM python:3.11 / USER appuser / COPY requirements.txt /app/requirements.txt / RUN pip install -r /app/requirements.txt (permission denied errors — what's wrong with the USER/COPY ordering and ownership?)

Mistake: switching to `appuser` before copying and installing means the copied files are owned by root and the unprivileged user cannot write to `/app` or install packages. The user may not even exist yet.

Fix. Create the file:

```bash
vi Containerfile
```

Put this inside:

```dockerfile
FROM python:3.11
RUN useradd -m appuser
WORKDIR /app
COPY requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt
USER appuser
```

Then build:

```bash
podman build -t fixed17 .
```

Do the privileged steps (create the user, copy files, install) as root first, then drop to `USER appuser` at the end for runtime. Switching users too early causes the permission denied errors.

---

## 18. FROM golang:1.22 / COPY . /src / WORKDIR /src / RUN go build -o app . / CMD ["/src/app"] (the final image is ~1 GB — how would a multi-stage build fix it?)

Mistake: the single stage keeps the whole Go toolchain and source in the final image, even though only the compiled binary is needed to run.

Fix. Create the file:

```bash
vi Containerfile
```

Put this inside:

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . /src
RUN go build -o app .

FROM alpine:3.20
COPY --from=build /src/app /app
CMD ["/app"]
```

Then build:

```bash
podman build -t fixed18 .
```

A multi stage build compiles in the heavy `golang` image, then copies only the finished binary into a tiny `alpine` final image. The build toolchain is discarded, so the image shrinks from about 1 GB to a few MB.

---

## 19. A pod is stuck Pending. Walk through the diagnosis with kubectl describe pod and identify the reason from the events (scheduling / resources / PVC / nodeSelector).

```bash
kubectl describe pod <pod> | grep -A10 Events
```

Read the Events. Common Pending reasons: `Insufficient cpu/memory` means requests are too high, so lower them. `node(s) didn't match nodeSelector` means no node has the label, so label a node. `pod has unbound immediate PersistentVolumeClaims` means the PVC is missing, so create it. `had taint` means add a toleration. The events line tells you which one it is.

---

## 20. Find a deployment from older tasks, remove a couple of spaces and try to deploy it. Identify the cause in the error messages and fix it.

Removing spaces breaks YAML indentation. Reproduce it, create the file:

```bash
vi bad.yaml
```

Put this inside (note the wrong indentation on `image`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
       image: nginx
```

Apply it to see the error, then fix the indentation:

```bash
kubectl apply -f bad.yaml     # error: converting YAML to JSON / did not find expected key
```

YAML is indentation sensitive, so a misaligned line produces a parse error like "did not find expected key". Fix by aligning `image` under `name` (both at the same depth) and reapply.

---

## 21. A pod is in ImagePullBackOff. Diagnose the cause (image typo, missing tag, private registry without pull secret) and fix it.

```bash
kubectl describe pod <pod> | grep -A5 Events
```

The events show `Failed to pull image`. If it is a typo or missing tag, fix the image name: `kubectl set image deployment/<dep> <container>=nginx:1.25`. If it is a private registry, create a docker-registry Secret and add it as `imagePullSecrets`. The exact message tells you which of the three it is.

---

## 22. A pod is in CrashLoopBackOff. Use kubectl logs --previous to read the last crash and fix the root cause.

```bash
kubectl logs <pod> --previous
```

`--previous` reads the logs of the container instance that just died, which shows the actual crash reason (a missing env var, a bad config path, a failed connection). You fix the root cause the log points to, for example add the missing variable or correct the command.

---

## 23. A pod's last state shows OOMKilled. Confirm it from kubectl get pod -o yaml / describe, then fix it (raise the memory limit or fix the app).

```bash
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState}'
kubectl describe pod <pod> | grep -i -A2 "Last State"
```

If `reason: OOMKilled` appears, the container hit its memory limit and was killed. Fix by raising the `resources.limits.memory` in the manifest, or by fixing the app if it is leaking memory.

---

## 24. A Deployment's pods never become Ready. Trace it to a failing readiness probe and correct the probe or the app.

```bash
kubectl describe pod <pod> | grep -i -A3 readiness
```

The events show `Readiness probe failed`. Usually the probe points at the wrong path or port, so correct the probe to match where the app actually listens, or fix the app if it truly is not serving. The pod stays out of the Service until the probe passes.

---

## 25. A Service returns nothing. Diagnose a Service/pod label mismatch using kubectl get endpoints.

```bash
kubectl get endpoints <svc>
```

If the Endpoints list is empty, the Service selector matches no pods. Compare `kubectl get svc <svc> -o jsonpath='{.spec.selector}'` with `kubectl get pods --show-labels` and make them agree. Once the labels match, the Endpoints populate and the Service starts responding.

---

## 26. Given a manifest where spec.selector.matchLabels doesn't match spec.template.metadata.labels, explain the error kubectl apply returns and fix it.

Reproduce, create the file:

```bash
vi mismatch.yaml
```

Put this inside (selector says `app: web`, template says `app: nginx`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
```

```bash
kubectl apply -f mismatch.yaml   # error: `selector` does not match template `labels`
```

A Deployment's selector must match its pod template labels, otherwise it could never own the pods it creates. Fix by making both `app: web` (or both the same value) and reapply.

---

## 27. Given a manifest whose resources.requests exceed any node's capacity, explain why the pod is Pending (Insufficient cpu/memory) and fix it.

Reproduce, create the file:

```bash
vi toobig.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: toobig
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests:
          cpu: "100"
          memory: "500Gi"
```

```bash
kubectl apply -f toobig.yaml
kubectl describe pod toobig | grep -A3 Events   # 0/1 nodes available: Insufficient cpu/memory
```

The scheduler cannot find a node with 100 CPUs and 500 GiB free, so the pod stays Pending. Fix by lowering the requests to something a node can satisfy, for example `cpu: "100m"` and `memory: "128Mi"`.

---

## 28. Given a pod mounting a PVC that doesn't exist, diagnose the FailedMount/Pending and create the PVC.

```bash
kubectl describe pod <pod> | grep -A3 Events   # persistentvolumeclaim "mypvc" not found
```

Create the missing PVC, open the file:

```bash
vi pvc.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 100Mi
```

```bash
kubectl apply -f pvc.yaml
```

The pod was Pending because it referenced a PVC that did not exist. Once the PVC is created and bound, the volume mounts and the pod starts.

---

## 29. A container based on ubuntu with no long-running command shows Completed/CrashLoopBackOff. Explain why and add a proper command.

Mistake: the ubuntu image has no long running process, so the container finishes immediately. Under a Deployment that gets restarted over and over, which shows as CrashLoopBackOff.

Fix. Create the file:

```bash
vi ubuntu-pod.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu
spec:
  containers:
    - name: ubuntu
      image: ubuntu:24.04
      command: ["sleep", "infinity"]
```

```bash
kubectl apply -f ubuntu-pod.yaml
```

A container only stays up while its main process runs. Adding a long running command like `sleep infinity` keeps it alive instead of completing and restarting.

---

## 30. Use kubectl get events --sort-by=.lastTimestamp (or kubectl events) to triage everything that happened to a pod over time.

```bash
kubectl get events --sort-by=.lastTimestamp
kubectl events --for pod/<pod>      # newer syntax, scoped to one pod
```

Sorting by timestamp gives you the full timeline of scheduling, pulling, starting, and any failures in order, which is the quickest way to see what happened to a pod over time.

---

## 31. Use kubectl exec -it to get a shell in a pod and debug DNS with nslookup and cat /etc/resolv.conf.

```bash
kubectl exec -it <pod> -- sh
# inside the pod:
cat /etc/resolv.conf
nslookup kubernetes.default
```

`/etc/resolv.conf` shows the cluster DNS server and search domains, and `nslookup` confirms whether names resolve. If resolution fails, the problem is DNS rather than the app.

---

## 32. Use an ephemeral debug container (kubectl debug -it <pod> --image=busybox --target=<container>) to troubleshoot a no-shell/distroless container.

```bash
kubectl debug -it <pod> --image=busybox --target=<container>
```

This attaches a temporary busybox container that shares the target container's process and network namespace, so you get shell tools against a distroless image that has no shell of its own. It disappears when you exit.

---

## 33. Spin up a throwaway kubectl run tmp --rm -it --image=busybox pod to test connectivity to a Service from inside the cluster.

```bash
kubectl run tmp --rm -it --image=busybox --restart=Never -- sh
# inside:
wget -qO- <svc>
nc -zv <svc> 80
```

A throwaway pod lets you test whether a Service is reachable from inside the cluster without installing anything. `--rm` deletes it when you exit.

---

## 34. A node shows NotReady. List what you'd check (kubelet, disk pressure, CNI) and inspect it on minikube with kubectl describe node and sudo minikube logs.

```bash
kubectl describe node minikube | grep -A10 Conditions
minikube status
sudo minikube logs | tail -50
```

Check the node Conditions for `DiskPressure`, `MemoryPressure`, or a `kubelet` that stopped posting status, and check that the CNI network plugin is running. On minikube, `minikube status` and the logs usually reveal a stopped kubelet or resource pressure.

---

## 35. A pod is stuck Terminating. Explain finalizers and the grace period, then force-delete it (--grace-period=0 --force) and state the risks.

```bash
kubectl delete pod <pod> --grace-period=0 --force
```

A pod normally waits out its termination grace period and any finalizers (cleanup hooks that must complete) before it is removed, which is why it can hang in Terminating. Force deleting skips both. The risk is that the container may still actually be running and its resources or storage not cleaned up, which for a stateful app can cause data corruption or a split brain.

---

## 36. Given a manifest with the wrong apiVersion/kind pairing (e.g. apps/v1 + Pod), explain the validation error and correct the group/version.

Reproduce, create the file:

```bash
vi wrongkind.yaml
```

Put this inside (Pod under apps/v1 is invalid):

```yaml
apiVersion: apps/v1
kind: Pod
metadata:
  name: p
spec:
  containers:
    - name: nginx
      image: nginx
```

```bash
kubectl apply -f wrongkind.yaml   # error: no matches for kind "Pod" in version "apps/v1"
```

Each kind belongs to a specific group and version. Pod is in the core group `v1`, while Deployment, StatefulSet, and DaemonSet are in `apps/v1`. Fix by setting `apiVersion: v1` for a Pod.

---

## 37. A pod fails with CreateContainerConfigError because a configMapKeyRef key is missing. Diagnose and fix.

```bash
kubectl describe pod <pod> | grep -A3 Events   # couldn't find key <key> in ConfigMap
```

The pod references a ConfigMap key that does not exist. Fix by adding the key to the ConfigMap, for example:

```bash
kubectl create configmap appcfg --from-literal=<key>=<value> --dry-run=client -o yaml | kubectl apply -f -
```

or correct the `configMapKeyRef` name in the pod spec to a key that actually exists. Then the container can build its config and start.

---

## 38. A pod can't mount a Secret that lives in a different namespace. Explain namespace scoping and fix it.

```bash
kubectl get secret <secret> -n <other-ns>
```

Secrets are namespace scoped, so a pod can only mount a Secret that lives in its own namespace. There is no cross namespace mount. Fix by creating the Secret in the pod's namespace:

```bash
kubectl get secret <secret> -n <other-ns> -o yaml \
  | sed 's/namespace: <other-ns>/namespace: <pod-ns>/' \
  | kubectl apply -n <pod-ns> -f -
```

Now the Secret exists alongside the pod and mounts correctly.

---

## 39. A rollout is stuck with ProgressDeadlineExceeded. Use kubectl describe deployment / rollout status to find the cause and remediate.

```bash
kubectl rollout status deployment/<dep>
kubectl describe deployment <dep> | grep -A5 Conditions
```

`ProgressDeadlineExceeded` means the new ReplicaSet did not become available in time, usually because of a bad image or a failing probe. Read the events on the new pods to find the real cause, fix it, or roll back with `kubectl rollout undo deployment/<dep>`.

---

## 40. Catch indentation/field typos (e.g. imagePullpolicy, misnested ports) before applying with kubectl apply --dry-run=server / --validate=true, then fix them.

```bash
kubectl apply -f manifest.yaml --dry-run=server
```

Server side dry run sends the manifest to the API for validation without creating anything, so it flags unknown fields like `imagePullpolicy` (the correct field is `imagePullPolicy`) and misnested keys. Fix the flagged field names and indentation, then apply for real.

---

## 41. An app's logs say it can't reach its database Service. Verify in order: pod running → Service exists → endpoints populated → DNS resolves → correct port — and identify the broken link.

```bash
kubectl get pods -l app=db                 # 1. DB pod running and Ready?
kubectl get svc db                         # 2. Service exists?
kubectl get endpoints db                   # 3. endpoints populated?
kubectl run tmp --rm -it --image=busybox --restart=Never -- nslookup db   # 4. DNS resolves?
kubectl get svc db -o jsonpath='{.spec.ports}'                            # 5. correct port?
```

Walk the chain in order and the first step that fails is the broken link. Empty endpoints point to a label mismatch, a DNS failure points to the wrong service name or namespace, and a wrong port points to a targetPort mismatch.

---

## 42. Read a container's state/lastState and restartCount with kubectl get pod -o jsonpath to determine the root cause of repeated restarts.

```bash
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].restartCount}{"\n"}'
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState}{"\n"}'
```

A high `restartCount` plus a `lastState` of `terminated` with a reason (like `OOMKilled` or a non zero exit code) tells you why the container keeps restarting, which points straight at the fix (more memory, corrected command, or missing config).

---

## 43. Compare OpenShift Route to Ingress.

Both expose a Service to the outside world over HTTP. An OpenShift Route is OpenShift's own built in object, works out of the box with the integrated router, and handles TLS termination simply per route. Ingress is the standard Kubernetes object, is portable across any cluster, but needs a separate ingress controller (like nginx or Traefik) installed before it does anything. So Route is simpler and integrated but OpenShift specific, while Ingress is portable and vendor neutral but requires you to run a controller. OpenShift supports both, and creating an Ingress there actually generates a Route under the hood.

---

## Cleanup at the end

```bash
podman rm -f $(podman ps -aq) 2>/dev/null
kubectl delete pod,deployment,svc,pvc,configmap,secret --all
minikube stop
```

