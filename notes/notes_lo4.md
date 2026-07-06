For the YAML ones the steps are always the same three: open the file with `vi`, type in the full content shown, then apply it with `kubectl apply -f`. Nothing is left for you to assemble.

> How to use `vi` quickly: type `vi name.yaml`, press `i` to start typing, paste or type the content, press `Esc`, then type `:wq` and Enter to save and quit.

---

## 0. One-time setup (run this first, once per session)

```bash
minikube start
kubectl get nodes
```

Wait until `kubectl get nodes` shows a node with status `Ready`. The cluster is now up and every command below will work. You only do this once at the start, not per question.

Handy check-and-clean commands you will reuse:

```bash
kubectl get all                  # see everything in the current namespace
kubectl describe <kind> <name>   # read events when something misbehaves
kubectl delete -f <file>.yaml    # remove what a manifest created
```

---

## 1. Create a Deployment named web running nginx:1.25 with 3 replicas using a single imperative kubectl create deployment command, then export its generated YAML to a file.

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl get deployment web
kubectl get deployment web -o yaml > web.yaml
```

One imperative command creates the Deployment, and `-o yaml` writes its live spec to `web.yaml`, which you can edit and reapply later.

---

## 2. Enable pulling images from DockerHub as an authenticated user to lower the chance of hitting the pull limit. What kind of resource do you need to create?

You create a **docker-registry Secret** and attach it so pods use it.

```bash
kubectl create secret docker-registry dockerhub \
  --docker-server=docker.io \
  --docker-username=YOURUSER \
  --docker-password=YOURPASS

kubectl patch serviceaccount default \
  -p '{"imagePullSecrets":[{"name":"dockerhub"}]}'
```

The resource is a `kubernetes.io/dockerconfigjson` Secret. Patching the default ServiceAccount makes every new pod use it automatically. Authenticated pulls have a higher rate limit than anonymous ones, so this lowers the chance of hitting the DockerHub pull limit.

---

## 3. Scale web from 3 to 5 replicas two different ways — once with kubectl scale, once by editing the manifest — and show the ReplicaSet that results.

Way 1, imperative:

```bash
kubectl scale deployment web --replicas=5
kubectl get rs -l app=web
```

Way 2, by editing the manifest. Create the file:

```bash
vi web-5.yaml
```

Put this inside:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 5
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
          image: nginx:1.25
```

Then apply and check:

```bash
kubectl apply -f web-5.yaml
kubectl get rs -l app=web
```

Both methods change the same field. The Deployment owns a ReplicaSet, and the ReplicaSet is what actually maintains the 5 pods.

---

## 4. Perform a rolling update of web from nginx:1.25 to nginx:1.27 and watch it with kubectl rollout status.

```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
kubectl get pods -l app=web
```

`set image` creates a new ReplicaSet, and `rollout status` blocks until the new pods are ready and the old ones are gone.

---

## 5. View a Deployment's rollout history and roll back to the previous revision. Explain what the CHANGE-CAUSE column shows and how to populate it.

```bash
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
kubectl annotate deployment/web kubernetes.io/change-cause="update to nginx 1.27" --overwrite
kubectl rollout history deployment/web
```

The CHANGE-CAUSE column shows why each revision was created. It is empty unless you set the `kubernetes.io/change-cause` annotation, which the third command does.

---

## 6. Set the update strategy to RollingUpdate with maxSurge: 1 and maxUnavailable: 0, and explain the effect on availability during an update.

Create the file:

```bash
vi web-strategy.yaml
```

Put this inside:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
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
          image: nginx:1.25
```

Then apply and check:

```bash
kubectl apply -f web-strategy.yaml
kubectl describe deployment web | grep -A2 StrategyType
```

`maxUnavailable: 0` means no old pod is removed until a new one is ready, and `maxSurge: 1` allows one extra pod during the update. The effect is zero downtime, at the cost of briefly running one more pod than the desired count.

---

## 7. Switch a Deployment's strategy to Recreate and describe a concrete scenario where this is required instead of RollingUpdate.

Create the file:

```bash
vi web-recreate.yaml
```

Put this inside:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  strategy:
    type: Recreate
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
          image: nginx:1.25
```

Then apply and check:

```bash
kubectl apply -f web-recreate.yaml
kubectl describe deployment web | grep StrategyType
```

Recreate kills all old pods first, then starts the new ones, so there is a short outage. You need it when old and new versions cannot run at the same time, for example when they share a single ReadWriteOnce volume, or a database schema that only one version understands.

---

## 8. Add CPU/memory requests and limits to a Deployment's container and verify they appear in the running pod spec.

Create the file:

```bash
vi web-resources.yaml
```

Put this inside:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
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
          image: nginx:1.25
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

Then apply and check:

```bash
kubectl apply -f web-resources.yaml
POD=$(kubectl get pod -l app=web -o name | head -1)
kubectl get $POD -o jsonpath='{.spec.containers[0].resources}'
echo
```

Requests are what the scheduler reserves, limits are the hard cap. The jsonpath output confirms they made it into the running pod spec.

---

## 9. Set revisionHistoryLimit: 3 and explain how it affects your ability to roll back and how many old ReplicaSets are kept.

Create the file:

```bash
vi web-history.yaml
```

Put this inside:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  revisionHistoryLimit: 3
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
          image: nginx:1.25
```

Then apply and check:

```bash
kubectl apply -f web-history.yaml
kubectl get rs -l app=web
```

This keeps the last 3 old ReplicaSets. You can only roll back to a revision that still exists, so a smaller number saves clutter but limits how far back you can undo.

---

## 10. Set a Deployment's image to a non-existent tag, observe the rollout get stuck, and explain why the old pods keep serving traffic.

```bash
kubectl set image deployment/web nginx=nginx:doesnotexist
kubectl rollout status deployment/web --timeout=30s
kubectl get pods -l app=web
```

The rollout waits for the new pod to become ready, which never happens because the image cannot be pulled (`ImagePullBackOff`). Because `maxUnavailable` protects the old pods, they are not removed until the new ones are ready, so they keep serving traffic the whole time. Roll back with `kubectl rollout undo deployment/web`.

---

## 11. Use a label selector to list only the pods belonging to one Deployment, and explain the Deployment → ReplicaSet → Pod selector relationship.

```bash
kubectl get pods -l app=web
```

A Deployment creates a ReplicaSet, the ReplicaSet creates Pods, and each layer selects the one below by label. The Deployment selects its ReplicaSet, and the ReplicaSet selects its Pods, all through matching labels like `app=web`.

---

## 12. Expose a Deployment with kubectl expose, then explain what object was created and how its selector was derived.

```bash
kubectl expose deployment web --port=80 --target-port=80
kubectl get svc web
kubectl get svc web -o jsonpath='{.spec.selector}'
echo
```

This creates a Service. Its selector is copied from the Deployment's pod template labels, so the Service automatically targets those pods.

---

## 13. Add a sidecar (second) container to a Deployment's pod template and explain how the two containers share the pod network and volumes.

Create the file:

```bash
vi web-sidecar.yaml
```

Put this inside:

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
          image: nginx:1.25
        - name: sidecar
          image: busybox
          command: ["sh","-c","while true; do echo tick; sleep 5; done"]
```

Then apply and check:

```bash
kubectl apply -f web-sidecar.yaml
kubectl get pods -l app=web
```

Both containers live in one pod, so they share the pod's network (same localhost and IP) and can share any volume mounted into both. That is how a sidecar reads or serves data for the main container.

---

## 14. Write a Deployment manifest for httpd:2.4 with 2 replicas, a named container port, and a nodeSelector; explain why it stays Pending if no node matches.

Create the file:

```bash
vi httpd.yaml
```

Put this inside:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd
spec:
  replicas: 2
  selector:
    matchLabels:
      app: httpd
  template:
    metadata:
      labels:
        app: httpd
    spec:
      nodeSelector:
        disktype: ssd
      containers:
        - name: httpd
          image: httpd:2.4
          ports:
            - name: http
              containerPort: 80
```

Then apply and check:

```bash
kubectl apply -f httpd.yaml
kubectl get pods -l app=httpd
kubectl describe pod -l app=httpd | grep -A3 Events
```

If no node has the label `disktype=ssd`, the scheduler cannot place the pods, so they stay Pending. To make it schedule, label a node with `kubectl label node minikube disktype=ssd`.

---

## 15. Create a bare Pod (no controller) running busybox that sleeps, then explain what happens when you delete it versus a Deployment-managed pod.

```bash
kubectl run bp --image=busybox --command -- sleep 3600
kubectl get pod bp
kubectl delete pod bp
kubectl get pod bp
```

A bare pod has no controller, so once you delete it (or it dies), nothing recreates it and it is gone. A Deployment managed pod is recreated by its ReplicaSet, which always restores the desired count.

---

## 16. Write a StatefulSet for redis:7 with 3 replicas plus a headless Service, and show the stable pod names and per-pod DNS records.

Create the file:

```bash
vi redis-sts.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
    - port: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7
          ports:
            - containerPort: 6379
```

Then apply and check:

```bash
kubectl apply -f redis-sts.yaml
kubectl get pods -l app=redis
kubectl run tmp --rm -it --image=busybox --restart=Never -- nslookup redis-0.redis.default.svc.cluster.local
```

Pods get stable names `redis-0`, `redis-1`, `redis-2`, and the headless Service gives each a DNS record like `redis-0.redis.default.svc.cluster.local`.

---

## 17. Create a DaemonSet running a busybox agent and explain why exactly one pod runs per node.

Create the file:

```bash
vi agent-ds.yaml
```

Put this inside:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: agent
spec:
  selector:
    matchLabels:
      app: agent
  template:
    metadata:
      labels:
        app: agent
    spec:
      containers:
        - name: agent
          image: busybox
          command: ["sh","-c","while true; do sleep 3600; done"]
```

Then apply and check:

```bash
kubectl apply -f agent-ds.yaml
kubectl get pods -l app=agent -o wide
```

A DaemonSet ensures exactly one pod per node, and it automatically adds a pod when a new node joins. It is used for node level agents like log or metrics collectors.

---

## 18. Create a Job using busybox/perl that computes something once and completes; show COMPLETIONS and read the result from logs.

```bash
kubectl create job pi --image=perl -- perl -Mbignum=bpi -wle "print bpi(20)"
kubectl wait --for=condition=complete job/pi --timeout=120s
kubectl get job pi
kubectl logs job/pi
```

A Job runs a pod to completion. COMPLETIONS reaching `1/1` means it finished, and the logs hold the computed value of pi.

---

## 19. Create a CronJob that prints the date every minute; show how to suspend it and how to list the Jobs it spawns.

```bash
kubectl create cronjob datejob --image=busybox --schedule="* * * * *" -- date
# wait about a minute, then:
kubectl get jobs
kubectl patch cronjob datejob -p '{"spec":{"suspend":true}}'
kubectl get cronjob datejob
```

A CronJob spawns a Job on each schedule tick. `kubectl get jobs` lists the Jobs it created, and `suspend: true` stops it firing without deleting it.

---

## 20. Manually run a Job from an existing CronJob (kubectl create job --from=cronjob/...) to test it on demand.

```bash
kubectl create job manual-run --from=cronjob/datejob
kubectl get job manual-run
kubectl logs job/manual-run
```

This creates a one off Job from the CronJob's template, which lets you test it immediately without waiting for the schedule.

---

## 21. Show that a StatefulSet creates and terminates pods in order, and contrast this with a Deployment.

```bash
kubectl get pods -l app=redis -w
# in another terminal: kubectl scale statefulset redis --replicas=0
# watch them terminate redis-2, then redis-1, then redis-0
```

A StatefulSet creates pods in order 0, 1, 2 and terminates them in reverse 2, 1, 0, waiting for each step. A Deployment creates and deletes its pods all at once with no ordering, because its pods are interchangeable.

---

## 22. Create a multi-container Pod where two containers share an emptyDir — one writes a file, the other reads it. Prove it's shared.

Create the file:

```bash
vi shared-pod.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared
spec:
  volumes:
    - name: shared
      emptyDir: {}
  containers:
    - name: writer
      image: busybox
      command: ["sh","-c","echo hello > /data/msg; sleep 3600"]
      volumeMounts:
        - name: shared
          mountPath: /data
    - name: reader
      image: busybox
      command: ["sh","-c","sleep 5; cat /data/msg; sleep 3600"]
      volumeMounts:
        - name: shared
          mountPath: /data
```

Then apply and check:

```bash
kubectl apply -f shared-pod.yaml
sleep 8
kubectl logs shared -c reader
```

The emptyDir is a shared scratch volume mounted into both containers, so the reader prints `hello`, the file the writer created.

---

## 23. List the StorageClasses, PVs and PVCs.

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc
```

StorageClasses define how volumes are provisioned, PVs are the actual pieces of storage, and PVCs are the claims that pods make against them.

---

## 24. Mount a ConfigMap as a volume so each key becomes a file, and verify the contents inside the pod.

First create the ConfigMap:

```bash
kubectl create configmap appcfg --from-literal=color=blue --from-literal=mode=dark
```

Create the file:

```bash
vi cm-pod.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cmpod
spec:
  volumes:
    - name: cfg
      configMap:
        name: appcfg
  containers:
    - name: app
      image: busybox
      command: ["sh","-c","sleep 3600"]
      volumeMounts:
        - name: cfg
          mountPath: /etc/appcfg
```

Then apply and check:

```bash
kubectl apply -f cm-pod.yaml
kubectl exec cmpod -- ls /etc/appcfg
kubectl exec cmpod -- cat /etc/appcfg/color
```

When you mount a ConfigMap as a volume, every key becomes a file whose contents are the value, so `color` and `mode` show up as files.

---

## 25. Mount a single ConfigMap key to a specific path using subPath; explain when it's needed and its update caveat.

Create the file:

```bash
vi subpath-pod.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: subpod
spec:
  volumes:
    - name: cfg
      configMap:
        name: appcfg
  containers:
    - name: app
      image: busybox
      command: ["sh","-c","sleep 3600"]
      volumeMounts:
        - name: cfg
          mountPath: /etc/app/color.conf
          subPath: color
```

Then apply and check:

```bash
kubectl apply -f subpath-pod.yaml
kubectl exec subpod -- cat /etc/app/color.conf
```

`subPath` drops one key into an existing directory without hiding the other files there, which is what you need when a config file must sit next to files the image already provides. The caveat is that subPath mounts do not update automatically when the ConfigMap changes, unlike a full volume mount.

---

## 26. Mount a Secret as a volume and verify the files contain decoded values with restrictive permissions.

First create the Secret:

```bash
kubectl create secret generic dbsecret --from-literal=password=s3cr3t
```

Create the file:

```bash
vi secret-pod.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secpod
spec:
  volumes:
    - name: sec
      secret:
        secretName: dbsecret
        defaultMode: 0400
  containers:
    - name: app
      image: busybox
      command: ["sh","-c","sleep 3600"]
      volumeMounts:
        - name: sec
          mountPath: /etc/secret
```

Then apply and check:

```bash
kubectl apply -f secret-pod.yaml
kubectl exec secpod -- cat /etc/secret/password
kubectl exec secpod -- ls -l /etc/secret/password
```

Mounted secret files hold the decoded value, not base64, and `defaultMode: 0400` makes them read only to the owner.

---

## 27. Show that scaling a StatefulSet creates one PVC per replica, then explain what happens to those PVCs when the StatefulSet is deleted.

Create the file:

```bash
vi redis-pvc-sts.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redisdata
spec:
  clusterIP: None
  selector:
    app: redisdata
  ports:
    - port: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redisdata
spec:
  serviceName: redisdata
  replicas: 3
  selector:
    matchLabels:
      app: redisdata
  template:
    metadata:
      labels:
        app: redisdata
    spec:
      containers:
        - name: redis
          image: redis:7
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Mi
```

Then apply and check:

```bash
kubectl apply -f redis-pvc-sts.yaml
kubectl get pvc
```

Each replica gets its own PVC from the `volumeClaimTemplates`, named `data-redisdata-0`, `data-redisdata-1`, and so on. When you delete the StatefulSet the PVCs are intentionally kept so data is not lost, and you delete them manually with `kubectl delete pvc ...` if you really want the storage gone.

---

## 28. Use an initContainer to pre-populate data into a shared volume before the main container reads it.

Create the file:

```bash
vi init-pod.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: initpod
spec:
  volumes:
    - name: shared
      emptyDir: {}
  initContainers:
    - name: init
      image: busybox
      command: ["sh","-c","echo seeded > /data/file"]
      volumeMounts:
        - name: shared
          mountPath: /data
  containers:
    - name: app
      image: busybox
      command: ["sh","-c","cat /data/file; sleep 3600"]
      volumeMounts:
        - name: shared
          mountPath: /data
```

Then apply and check:

```bash
kubectl apply -f init-pod.yaml
sleep 5
kubectl logs initpod
```

The initContainer runs to completion first and writes into the shared volume, so the main container finds the data already there and prints `seeded`.

---

## 29. Set readOnly: true on a volume mount and prove writes are rejected.

Create the file:

```bash
vi ro-pod.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ropod
spec:
  volumes:
    - name: cfg
      configMap:
        name: appcfg
  containers:
    - name: app
      image: busybox
      command: ["sh","-c","sleep 3600"]
      volumeMounts:
        - name: cfg
          mountPath: /etc/appcfg
          readOnly: true
```

Then apply and check:

```bash
kubectl apply -f ro-pod.yaml
kubectl exec ropod -- sh -c "echo x > /etc/appcfg/new"
```

`readOnly: true` mounts the volume so any write attempt fails with a read-only file system error.

---

## 30. Set a sizeLimit on an emptyDir and explain what happens if the container exceeds it.

Create the file:

```bash
vi sizelimit-pod.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sizepod
spec:
  volumes:
    - name: scratch
      emptyDir:
        sizeLimit: 100Mi
  containers:
    - name: app
      image: busybox
      command: ["sh","-c","sleep 3600"]
      volumeMounts:
        - name: scratch
          mountPath: /scratch
```

Then apply and check:

```bash
kubectl apply -f sizelimit-pod.yaml
kubectl get pod sizepod
```

If the container writes more than the limit into the emptyDir, the pod is evicted for exceeding its ephemeral storage. It caps how much scratch space a pod can use.

---

## 31. Create a generic Secret from literals for a DB username/password, then show the stored values are base64-encoded, not encrypted.

```bash
kubectl create secret generic dblogin \
  --from-literal=username=admin --from-literal=password=s3cr3t
kubectl get secret dblogin -o jsonpath='{.data.password}'
echo
kubectl get secret dblogin -o jsonpath='{.data.password}' | base64 -d
echo
```

The stored value is only base64 encoded, which is encoding, not encryption. Anyone who can read the Secret can decode it, so RBAC access control is what actually protects it.

---

## 32. Create a Secret from files (--from-file) holding a certificate and key, and identify the resulting keys.

```bash
openssl req -x509 -newkey rsa:2048 -nodes -keyout tls.key -out tls.crt -days 1 -subj "/CN=test"
kubectl create secret generic tlsfiles --from-file=tls.crt --from-file=tls.key
kubectl describe secret tlsfiles
```

Each file becomes a key named after the filename, so you get keys `tls.crt` and `tls.key`.

---

## 33. Create a kubernetes.io/tls typed Secret from a cert/key pair and explain where it's consumed.

```bash
kubectl create secret tls mytls --cert=tls.crt --key=tls.key
kubectl get secret mytls -o jsonpath='{.type}'
echo
```

A `kubernetes.io/tls` Secret always holds keys `tls.crt` and `tls.key`. It is consumed by Ingress controllers and by pods that serve HTTPS, which read the certificate and key from it.

---

## 34. Mount a Secret as a volume and explain the security trade-offs versus env vars.

Create the file:

```bash
vi sec-vol-pod.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secvol
spec:
  volumes:
    - name: sec
      secret:
        secretName: dblogin
  containers:
    - name: app
      image: busybox
      command: ["sh","-c","sleep 3600"]
      volumeMounts:
        - name: sec
          mountPath: /etc/secret
```

Then apply and check:

```bash
kubectl apply -f sec-vol-pod.yaml
kubectl exec secvol -- ls /etc/secret
```

A mounted Secret can be updated in place and the files refresh without a restart, and it is less likely to leak because environment variables can show up in logs, in `kubectl describe`, or be inherited by child processes. The downside is that reading a file is slightly more work than reading an env var. Env vars are simpler but riskier for sensitive values.

---

## 35. Use envFrom with a secretRef to load every key of a Secret as env vars at once.

Create the file:

```bash
vi envfrom-pod.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envpod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh","-c","env; sleep 3600"]
      envFrom:
        - secretRef:
            name: dblogin
```

Then apply and check:

```bash
kubectl apply -f envfrom-pod.yaml
sleep 3
kubectl logs envpod | grep -E "username|password"
```

`envFrom` with a `secretRef` loads every key of the Secret as an environment variable at once, so you do not map each key by hand.

---

## 36. Create a docker-registry (image-pull) Secret and reference it via imagePullSecrets; explain when it's required.

First create the Secret:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=u --docker-password=p
```

Create the file:

```bash
vi pullsecret-pod.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privatepod
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: registry.example.com/myapp:1.0
```

Then apply and check:

```bash
kubectl apply -f pullsecret-pod.yaml
kubectl describe pod privatepod | grep -A1 "Image Pull"
```

You need this when pulling from a private registry that requires authentication. Without the `imagePullSecrets` reference the pull fails with an auth error.

---

## 37. Create a Secret with several keys and selectively mount only one of them into a pod.

First create the Secret:

```bash
kubectl create secret generic multikey \
  --from-literal=user=admin --from-literal=password=s3cr3t --from-literal=token=abc123
```

Create the file:

```bash
vi selectkey-pod.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: onekey
spec:
  volumes:
    - name: sec
      secret:
        secretName: multikey
        items:
          - key: password
            path: only-password
  containers:
    - name: app
      image: busybox
      command: ["sh","-c","sleep 3600"]
      volumeMounts:
        - name: sec
          mountPath: /etc/secret
```

Then apply and check:

```bash
kubectl apply -f selectkey-pod.yaml
kubectl exec onekey -- ls /etc/secret
```

Using `items` you mount just the chosen key, so only `only-password` appears as a file and the other keys are left out.

---

## 38. Create a ClusterIP Service for a Deployment and resolve it by DNS from another pod (nslookup <svc>.<ns>.svc.cluster.local).

```bash
kubectl create deployment web2 --image=nginx
kubectl expose deployment web2 --port=80
kubectl run tmp --rm -it --image=busybox --restart=Never -- nslookup web2.default.svc.cluster.local
```

A ClusterIP Service gets a stable virtual IP and a DNS name `<svc>.<ns>.svc.cluster.local`, which any pod in the cluster can resolve.

---

## 39. Create a NodePort Service and reach it via minikube service <svc> --url and $(minikube ip):<nodePort>.

```bash
kubectl expose deployment web2 --name=web2-np --type=NodePort --port=80
minikube service web2-np --url
NODEPORT=$(kubectl get svc web2-np -o jsonpath='{.spec.ports[0].nodePort}')
curl $(minikube ip):$NODEPORT
```

A NodePort opens the same high port on every node, so you reach the service through the node IP plus that port.

---

## 40. Create a LoadBalancer Service, explain why EXTERNAL-IP is <pending> on minikube, and obtain access with minikube tunnel.

```bash
kubectl expose deployment web2 --name=web2-lb --type=LoadBalancer --port=80
kubectl get svc web2-lb
# in a second terminal, leave this running:
minikube tunnel
# back in the first terminal:
kubectl get svc web2-lb
```

EXTERNAL-IP stays `<pending>` because minikube has no real cloud load balancer to provision one. `minikube tunnel` fakes it locally and assigns an external IP you can then curl.

---

## 41. Create a headless Service (clusterIP: None) for a StatefulSet and show it returns per-pod A records instead of a single VIP.

The headless Service was already created in question 16. Verify it returns per-pod records:

```bash
kubectl run tmp --rm -it --image=busybox --restart=Never -- nslookup redis
```

With `clusterIP: None` there is no single virtual IP. DNS instead returns one A record per pod, which is how StatefulSet clients address individual pods.

---

## 42. Inspect a Service's Endpoints/EndpointSlice and explain how they're populated from the pod selector.

```bash
kubectl get endpoints web2
kubectl get endpointslices -l kubernetes.io/service-name=web2
```

Endpoints list the pod IPs behind a Service. They are populated automatically from the Service's selector: every ready pod that matches the labels gets added.

---

## 43. Demonstrate cross-namespace access using the FQDN, and show the short name fails from another namespace.

```bash
kubectl create namespace other
kubectl run tmp -n other --rm -it --image=busybox --restart=Never -- sh -c \
  "nslookup web2.default.svc.cluster.local; echo ---; nslookup web2"
```

The full name resolves from any namespace, but the short name `web2` fails from the `other` namespace because the short name only resolves within the same namespace.

---

## 44. Compare kubectl port-forward to a Service versus to a Pod — explain the difference and when each fits.

```bash
kubectl port-forward svc/web2 8080:80
# or, targeting one exact pod:
# kubectl port-forward pod/<podname> 8080:80
```

Forwarding to a Service picks one backing pod and tunnels to it, which is handy when you do not know or care which pod. Forwarding to a Pod hits that exact pod, which is what you want when debugging one specific instance.

---

## 45. Explain the difference between a Service's port, targetPort, and nodePort.

`port` is the port the Service listens on inside the cluster. `targetPort` is the container port it forwards traffic to. `nodePort` is the port opened on every node for external access, and it only exists for NodePort or LoadBalancer type Services. You can view all three with:

```bash
kubectl get svc web2-np -o yaml | grep -A5 ports
```

---

## 46. Verify connectivity to a ClusterIP Service from a throwaway debug pod (kubectl run tmp --rm -it --image=busybox -- sh) with wget/nc.

```bash
kubectl run tmp --rm -it --image=busybox --restart=Never -- sh -c "wget -qO- web2; nc -zv web2 80"
```

A throwaway busybox pod lets you test service reachability from inside the cluster without installing anything permanent.

---

## 47. Create two Deployments and one Service that load-balances across both by sharing a common label; prove requests hit both.

Create the file:

```bash
vi shared-svc.yaml
```

Put this inside:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: shared
      ver: v1
  template:
    metadata:
      labels:
        app: shared
        ver: v1
    spec:
      containers:
        - name: web
          image: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: shared
      ver: v2
  template:
    metadata:
      labels:
        app: shared
        ver: v2
    spec:
      containers:
        - name: web
          image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: shared-svc
spec:
  selector:
    app: shared
  ports:
    - port: 80
```

Then apply and check:

```bash
kubectl apply -f shared-svc.yaml
kubectl get endpoints shared-svc
```

Because the Service selects `app=shared`, and pods from both Deployments carry that label, its Endpoints include one pod IP from each Deployment, so requests are balanced across both.

---

## 48. Systematically diagnose why a pod can't reach a Service: DNS → endpoints → selector labels → readiness → port.

```bash
kubectl run tmp --rm -it --image=busybox --restart=Never -- nslookup shared-svc   # DNS resolves?
kubectl get endpoints shared-svc                                                  # endpoints populated?
kubectl get pods --show-labels                                                     # labels match selector?
kubectl get pods                                                                   # pods Ready?
kubectl get svc shared-svc -o yaml | grep -A5 ports                                # correct port?
```

Check in that order, and the first step that fails is the broken link. A Service with no Endpoints almost always means a selector and label mismatch or no ready pods.

---

## 49. Expose a service with a NodePort and verify it works.

```bash
kubectl expose deployment web2 --name=verify-np --type=NodePort --port=80
NODEPORT=$(kubectl get svc verify-np -o jsonpath='{.spec.ports[0].nodePort}')
curl $(minikube ip):$NODEPORT
```

The NodePort maps an external node port to the service, and the curl output (the nginx welcome page) confirms it answers.

---

## 50. Add an HTTP livenessProbe hitting / on port 80 to an nginx Deployment and verify via kubectl describe pod.

Create the file:

```bash
vi liveness.yaml
```

Put this inside:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: liveweb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: liveweb
  template:
    metadata:
      labels:
        app: liveweb
    spec:
      containers:
        - name: nginx
          image: nginx
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
```

Then apply and check:

```bash
kubectl apply -f liveness.yaml
kubectl describe pod -l app=liveweb | grep -i liveness
```

The liveness probe restarts the container if `/` on port 80 stops responding. The describe output shows the Liveness line.

---

## 51. Add a tcpSocket readiness probe to a redis container on port 6379 and verify it.

Create the file:

```bash
vi readiness.yaml
```

Put this inside:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redisready
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redisready
  template:
    metadata:
      labels:
        app: redisready
    spec:
      containers:
        - name: redis
          image: redis:7
          readinessProbe:
            tcpSocket:
              port: 6379
            initialDelaySeconds: 5
            periodSeconds: 10
```

Then apply and check:

```bash
kubectl apply -f readiness.yaml
kubectl get pods -l app=redisready
kubectl describe pod -l app=redisready | grep -i readiness
```

The readiness probe just checks that port 6379 accepts a TCP connection, and the pod is only marked `READY 1/1` when it does.

---

## 52. Configure a startup probe with a failureThreshold * periodSeconds budget for a ~2-minute boot and justify the numbers.

Create the file:

```bash
vi startup.yaml
```

Put this inside:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: slowboot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: slowboot
  template:
    metadata:
      labels:
        app: slowboot
    spec:
      containers:
        - name: nginx
          image: nginx
          startupProbe:
            httpGet:
              path: /
              port: 80
            failureThreshold: 30
            periodSeconds: 5
```

Then apply and check:

```bash
kubectl apply -f startup.yaml
kubectl describe pod -l app=slowboot | grep -i startup
```

`failureThreshold * periodSeconds` is the total time allowed to start, here 30 times 5 equals 150 seconds, which comfortably covers a roughly 2 minute boot. The startup probe holds off the liveness probe so a slow starting app is not killed before it is ready.

---

## 53. Explain the default behavior when no probes are defined at all.

With no probes defined, Kubernetes assumes the container is alive as soon as the process runs and Ready as soon as it starts. It never restarts the container for being unresponsive and sends traffic to it immediately, even if the app inside is not actually ready to serve. Probes exist precisely to fix that blind spot. There is nothing to apply for this one, it is a written answer.

---

## Cleanup at the end

```bash
kubectl delete deployment,statefulset,daemonset,job,cronjob,pod,svc,configmap,secret --all
minikube stop
```
