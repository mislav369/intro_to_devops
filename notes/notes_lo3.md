# LO3 — Application delivery using containers, network architecture, and component security

For the file based ones (compose.yaml, nginx.conf, pod YAML) the steps are always the same three: open the file with `vi`, type in the full content shown, then run the command that uses it. 

> How to use `vi` quickly: type `vi name.yaml`, press `i` to start typing, paste or type the content, press `Esc`, then type `:wq` and Enter to save and quit.

---

## 0. One-time setup (run once at the start)

Podman is already installed on the exam VM. Quick check that it and compose work:

```bash
podman version
podman compose version    # if this errors, try: podman-compose version
```

Handy check-and-clean commands you will reuse:

```bash
podman ps -a                 # list all containers
podman logs <name>           # see why a container exited
podman rm -f <name>          # remove a container
podman volume rm <name>      # remove a volume
```

---

## 1. Create a pod with podman pod create (publishing a port), add an app container and a database container to it, and show they reach each other over localhost (shared network namespace).

```bash
podman pod create --name mypod -p 8080:80
podman run -d --pod mypod --name db -e POSTGRES_PASSWORD=secret docker.io/library/postgres:16
podman run -d --pod mypod --name app docker.io/library/nginx
# prove they share localhost:
podman exec db sh -c "apt-get update -qq && apt-get install -y netcat-openbsd -qq; nc -zv localhost 5432"
```

All containers in a pod share one network namespace, so they reach each other on `localhost` / `127.0.0.1`. Note the port is published on the pod at create time, not on the individual container.

---

## 2. Create a user-defined network, run an app container and a postgres container on it, and prove the app resolves the database by container name (podman DNS on custom networks).

```bash
podman network create mynet
podman run -d --network mynet --name db -e POSTGRES_PASSWORD=secret docker.io/library/postgres:16
podman run -d --network mynet --name app docker.io/library/alpine sleep 1d
podman exec app getent hosts db     # resolves "db" to its IP
```

On a user defined network Podman runs an internal DNS server, so containers can find each other by name. The default network does not provide name resolution.

---

## 3. Deploy a two-tier stack of your choice that is not Drupal/Joomla/WordPress (e.g. Ghost + MySQL, Gitea + PostgreSQL, or Redmine + PostgreSQL) and complete its first-run setup.

Example: Gitea + PostgreSQL.

```bash
podman network create giteanet
podman run -d --network giteanet --name gitea-db \
  -e POSTGRES_USER=gitea -e POSTGRES_PASSWORD=gitea -e POSTGRES_DB=gitea \
  docker.io/library/postgres:16
podman run -d --network giteanet --name gitea -p 3000:3000 \
  docker.io/gitea/gitea:latest
```

Open `http://localhost:3000` and complete the setup form: database type PostgreSQL, host `gitea-db:5432`, user `gitea`, password `gitea`, database `gitea`. The app reaches the database by container name over the shared network.

---

## 4. Persist the database's data in a named volume so it survives podman rm + re-creation of the DB container; prove the data is still there afterward.

```bash
podman volume create pgdata
podman run -d --name db -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret docker.io/library/postgres:16
podman exec db psql -U postgres -c "CREATE TABLE t(x int); INSERT INTO t VALUES (1);"
podman rm -f db
podman run -d --name db -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret docker.io/library/postgres:16
podman exec db psql -U postgres -c "SELECT * FROM t;"   # row is still there
```

A named volume lives outside the container lifecycle, so removing and recreating the container does not touch the data.

---

## 5. Use a podman secret (podman secret create + --secret) to pass the database password instead of --env; explain the security benefit.

```bash
printf "my-secret-pw" | podman secret create db_pass -
podman run -d --name db \
  --secret db_pass,type=env,target=POSTGRES_PASSWORD \
  docker.io/library/postgres:16
# file form instead of env:
# podman run -d --name db --secret db_pass \
#   -e POSTGRES_PASSWORD_FILE=/run/secrets/db_pass docker.io/library/postgres:16
```

The secret is stored in Podman's encrypted secret store and injected only at runtime. Unlike `--env`, the password does not show up in `podman inspect`, in the image, or in the plain environment listing, so it is much less likely to leak.

---

## 6. Write a podman compose file (compose.yaml) for an app + database, bring it up with podman compose up -d, and show both services running.

Create the file:

```bash
vi compose.yaml
```

Put this inside:

```yaml
services:
  db:
    image: docker.io/library/postgres:16
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_USER: appuser
      POSTGRES_DB: appdb
  app:
    image: docker.io/library/nginx
    ports:
      - "8080:80"
    depends_on:
      - db
```

Then bring it up and check:

```bash
podman compose up -d
podman compose ps      # both services show as running
```

---

## 7. In that compose file, keep the database on an internal network with no published port and expose only the app; prove the DB isn't reachable from the host.

Create the file:

```bash
vi compose.yaml
```

Put this inside:

```yaml
services:
  db:
    image: docker.io/library/postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    networks:
      - backend
    # no ports: block, so the host cannot reach it
  app:
    image: docker.io/library/nginx
    ports:
      - "8080:80"
    networks:
      - backend
networks:
  backend:
```

Then bring it up and prove it:

```bash
podman compose up -d
podman compose exec app getent hosts db   # app reaches db by name
nc -zv localhost 5432                      # from host: connection refused
```

The database publishes no host port, so only containers on the shared `backend` network can reach it. The host has no route to it.

---

## 8. Add a compose healthcheck plus depends_on: condition: service_healthy so the app waits for the database; demonstrate the startup ordering.

Create the file:

```bash
vi compose.yaml
```

Put this inside:

```yaml
services:
  db:
    image: docker.io/library/postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
  app:
    image: docker.io/library/nginx
    depends_on:
      db:
        condition: service_healthy
```

Then bring it up and watch the ordering:

```bash
podman compose up -d
podman compose ps
```

The app container will not start until the database healthcheck reports healthy. Watching the startup you see the app wait for the DB to pass its check first.

---

## 9. Use an env_file: in compose for the database credentials instead of inline values.

First create the env file:

```bash
vi db.env
```

Put this inside:

```
POSTGRES_PASSWORD=secret
POSTGRES_USER=appuser
POSTGRES_DB=appdb
```

Then create the compose file:

```bash
vi compose.yaml
```

Put this inside:

```yaml
services:
  db:
    image: docker.io/library/postgres:16
    env_file:
      - db.env
```

Then bring it up:

```bash
podman compose up -d
podman compose ps
```

Credentials live in a separate file instead of inline in the compose file, so you can keep them out of version control with a `.gitignore`.

---

## 10. Scale one compose service to multiple replicas (podman compose up --scale app=3) and explain how requests are distributed.

Using the compose.yaml from question 6 (but remove the fixed host port on `app` first, since only one container can own a host port):

```bash
podman compose up -d --scale app=3
podman compose ps      # three app replicas
```

This starts three replicas of `app`. Because they share a service name, the internal DNS returns multiple addresses and requests are spread across the replicas by round robin. In practice you put a reverse proxy in front so a single host port can fan out to all three.

---

## 11. Mount a bind mount for app configuration and a named volume for data in the same container, and explain why you'd use each.

```bash
mkdir -p ./config
echo "setting=value" > ./config/app.conf
podman volume create appdata
podman run -d --name app \
  -v ./config:/etc/app:ro \
  -v appdata:/var/lib/app \
  docker.io/library/nginx
```

The bind mount points at a host directory, which is ideal for configuration you want to edit directly from the host (read only here so the container cannot change it). The named volume is Podman managed storage, better for durable application data you want to persist and back up independently of any host path.

---

## 12. Back up a named volume to a tarball using a helper alpine/busybox container that mounts the volume, then restore it into a fresh volume.

```bash
# backup
mkdir -p ./backup
podman run --rm -v pgdata:/data:ro -v ./backup:/backup docker.io/library/alpine \
  tar czf /backup/pgdata.tar.gz -C /data .
# restore into a fresh volume
podman volume create pgdata_new
podman run --rm -v pgdata_new:/data -v ./backup:/backup docker.io/library/alpine \
  tar xzf /backup/pgdata.tar.gz -C /data
```

A throwaway helper container mounts the volume plus a host backup folder and tars the contents out. Restore mounts a new empty volume and untars into it.

---

## 13. Put an nginx reverse-proxy container in front of an app container on a shared network and route host traffic through nginx to the app.

First create the nginx config file:

```bash
vi nginx.conf
```

Put this inside:

```
server {
  listen 80;
  location / {
    proxy_pass http://app:80;
  }
}
```

Then run both containers on a shared network:

```bash
podman network create webnet
podman run -d --network webnet --name app docker.io/library/httpd
podman run -d --network webnet --name proxy -p 8080:80 \
  -v ./nginx.conf:/etc/nginx/conf.d/default.conf:ro docker.io/library/nginx
curl http://localhost:8080
```

Both containers are on the same network, so nginx resolves `app` by name. The host reaches nginx on port 8080 and nginx forwards the request to the app. The app itself never needs a published port.

---

## 14. Add a database-admin container (e.g. adminer or pgadmin) to the network and use it to inspect the database.

```bash
podman run -d --network mynet --name adminer -p 8081:8080 docker.io/library/adminer
```

Open `http://localhost:8081`, choose system PostgreSQL, server `db`, and log in with the DB user and password. Adminer sits on the same network, so it reaches the database by name and gives you a web UI to browse tables.

---

## 15. Generate Kubernetes YAML from a running pod with podman generate kube and explain how it bridges LO3 to LO4.

```bash
podman generate kube mypod > mypod.yaml
cat mypod.yaml
```

This exports the pod and its containers as a Kubernetes manifest. It bridges LO3 to LO4 because you can build and test a stack locally with Podman, then take the generated YAML and run the same workload on a real Kubernetes cluster.

---

## 16. Run a pod from a Kubernetes YAML with podman kube play, then tear it down with podman kube play --down.

Create the file:

```bash
vi mypod.yaml
```

Put this inside:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webpod
spec:
  containers:
    - name: web
      image: docker.io/library/nginx
      ports:
        - containerPort: 80
          hostPort: 8090
```

Then play it and tear it down:

```bash
podman kube play mypod.yaml
podman ps
podman kube play --down mypod.yaml
```

`kube play` creates the pod and containers described in the manifest. `--down` removes everything the manifest created.

---

## 17. Set per-service CPU/memory limits in a compose file and verify them with podman stats.

Create the file:

```bash
vi compose.yaml
```

Put this inside:

```yaml
services:
  app:
    image: docker.io/library/nginx
    ports:
      - "8080:80"
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
```

Then bring it up and verify:

```bash
podman compose up -d
podman stats     # shows CPU % and MEM usage / limit per container
```

`podman stats` shows live usage against the limit, confirming the caps are applied.

---

## 18. Inspect a pod with podman pod inspect and show which Linux namespaces the containers share (network, IPC) and which they don't (PID by default).

```bash
podman pod inspect mypod
podman pod inspect mypod --format '{{.SharedNamespaces}}'
```

Look at `SharedNamespaces`. By default containers in a pod share the network, IPC, and UTS namespaces, which is why they talk over localhost and share a hostname. They do not share the PID namespace by default, so each container sees only its own processes unless you create the pod with `--share=pid`.

---

## 19. Demonstrate name-based discovery failing when two containers are on different networks, then fix it by attaching them to a common network.

```bash
podman network create net1
podman network create net2
podman run -d --network net1 --name a docker.io/library/alpine sleep 1d
podman run -d --network net2 --name b docker.io/library/alpine sleep 1d
podman exec a ping -c1 b        # fails: different networks, no shared DNS
podman network connect net1 b   # attach b to net1
podman exec a ping -c1 b        # now resolves and works
```

Name resolution only works between containers on the same user defined network. Attaching both to a common network fixes it.

---

## 20. Attach a single container to two networks (a frontend and a backend) and explain the segmentation benefit.

```bash
podman network create frontend
podman network create backend
podman run -d --name app --network frontend docker.io/library/nginx
podman network connect backend app
podman inspect app --format '{{.NetworkSettings.Networks}}'
```

The app sits on the frontend network (facing users) and the backend network (talking to the database), while the database stays only on backend. The segmentation benefit is that the database is never exposed on the frontend network, so traffic is isolated and the attack surface is smaller.

---

## 21. Use podman compose logs and podman compose ps to observe and troubleshoot a multi-service stack.

```bash
podman compose ps          # status of every service
podman compose logs        # aggregated logs from all services
podman compose logs app    # just one service
```

`ps` tells you which service is up or has exited, and `logs` shows why a service failed, so together they let you trace a broken multi service stack.

---

## 22. Share configuration across two replicas of the same app via a shared named volume, and explain a risk of shared read-write storage.

Create the file:

```bash
vi compose.yaml
```

Put this inside:

```yaml
services:
  app:
    image: docker.io/library/nginx
    volumes:
      - shared:/shared
volumes:
  shared:
```

Then bring it up scaled to two replicas:

```bash
podman compose up -d --scale app=2
podman compose ps
```

Both replicas mount the same volume, so they see the same files. The risk is that shared read write storage has no coordination between replicas, so two containers writing the same file at once can cause race conditions or data corruption. Shared volumes are safest when the data is read only or writes are carefully partitioned.

---

## 23. Tear down a stack with podman compose down, then explain what happens to named volumes and how --volumes changes that.

```bash
podman compose down             # removes containers and networks, keeps named volumes
podman compose down --volumes   # also removes the named volumes, deleting the data
```

By default `down` leaves named volumes in place, so your data survives a teardown and is still there next time you bring the stack up. Adding `--volumes` (or `-v`) also deletes the named volumes declared in the file, wiping the data with them.

---

## Cleanup at the end

```bash
podman compose down --volumes 2>/dev/null
podman rm -f $(podman ps -aq) 2>/dev/null
podman pod rm -f $(podman pod ls -q) 2>/dev/null
podman network prune -f
podman volume prune -f
```
