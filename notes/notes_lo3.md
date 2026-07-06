Create a pod with podman pod create (publishing a port), add an app container and a database container to
it, and show they reach each other over localhost (shared network namespace).

```bash
podman pod create --name mypod -p 8080:80
podman run -d --pod mypod --name db -e POSTGRES_PASSWORD=secret docker.io/library/postgres:16
podman run -d --pod mypod --name app docker.io/library/nginx
# prove they share localhost:
podman exec db sh -c "apt-get update -qq && apt-get install -y netcat-openbsd -qq; nc -zv localhost 5432"
```

All containers in a pod share one network namespace, so they reach each other on `localhost` / `127.0.0.1`. Note the port is published on the pod at create time, not on the individual container.

---

Create a user-defined network, run an app container and a postgres container on it, and prove the app
resolves the database by container name (podman DNS on custom networks).

```bash
podman network create mynet
podman run -d --network mynet --name db -e POSTGRES_PASSWORD=secret docker.io/library/postgres:16
podman run -d --network mynet --name app docker.io/library/alpine sleep 1d
podman exec app getent hosts db     # resolves "db" to its IP
```

---

On a user defined network Podman runs an internal DNS server, so containers can find each other by name. The default network does not provide name resolution.

Deploy a two-tier stack of your choice that is not Drupal/Joomla/WordPress (e.g. Ghost + MySQL, Gitea +
PostgreSQL, or Redmine + PostgreSQL) and complete its first-run setup.

Persist the database's data in a named volume so it survives podman rm + re-creation of the DB container;
prove the data is still there afterward.


Use a podman secret (podman secret create + --secret) to pass the database password instead of --env;
explain the security benefit.

Write a podman compose file (compose.yaml) for an app + database, bring it up with podman compose up -
d, and show both services running.

In that compose file, keep the database on an internal network with no published port and expose only the
app; prove the DB isn't reachable from the host.

Add a compose healthcheck plus depends_on: condition: service_healthy so the app waits for the database;
demonstrate the startup ordering.

Use an env_file: in compose for the database credentials instead of inline values.

Scale one compose service to multiple replicas (podman compose up --scale app=3) and explain how
requests are distributed.

Mount a bind mount for app configuration and a named volume for data in the same container, and explain
why you'd use each.

Back up a named volume to a tarball using a helper alpine/busybox container that mounts the volume, then
restore it into a fresh volume.

Put an nginx reverse-proxy container in front of an app container on a shared network and route host traffic
through nginx to the app.

Add a database-admin container (e.g. adminer or pgadmin) to the network and use it to inspect the
database.

Generate Kubernetes YAML from a running pod with podman generate kube and explain how it bridges LO3
to LO4.

Run a pod from a Kubernetes YAML with podman kube play, then tear it down with podman kube play --
down.

Set per-service CPU/memory limits in a compose file and verify them with podman stats.

Inspect a pod with podman pod inspect and show which Linux namespaces the containers share (network,
IPC) and which they don't (PID by default).

Demonstrate name-based discovery failing when two containers are on different networks, then fix it by
attaching them to a common network.

Attach a single container to two networks (a frontend and a backend) and explain the segmentation benefit.

Use podman compose logs and podman compose ps to observe and troubleshoot a multi-service stack.

Share configuration across two replicas of the same app via a shared named volume, and explain a risk of
shared read-write storage.

Tear down a stack with podman compose down, then explain what happens to named volumes and how --
volumes changes that.
