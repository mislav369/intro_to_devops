Analyze the networking model of Kubernetes versus Docker's network

<br>

In Docker the container gets an IP on a local bridge and you have to publish ports with -p and use NAT to reach it, and by default it all lives on one host. Kubernetes gives every pod its own cluster wide IP and lets any pod reach any pod directly with no NAT, using a CNI plugin like Calico. Since pod IPs change, you put a Service in front for a stable IP plus load balancing, and CoreDNS handles names. So Docker is single host with NAT and port mapping, Kubernetes is multi host with real pod IPs and Services for discovery.

<br>
<br>

Evaluate Kubernetes storage abstraction (CSI, StorageClasses, dynamic provisioning) against Podman’s
storage plugins. Which is more flexible for stateful workloads, and why?

<br>

Kubernetes is more flexible. An app asks for storage with a PVC, a StorageClass provisions it automatically, and CSI lets it sit on any backend like EBS, Ceph, or NFS, plus StatefulSets give each replica its own volume that follows it when a pod moves nodes. Podman only has local named volumes and bind mounts on a single host, with no dynamic provisioning. So for stateful workloads Kubernetes wins because storage can follow the workload across the cluster.

<br>
<br>

Evaluate the operator pattern (CRDs + controllers) for managing stateful services versus using only k8s
manifests, in terms of control and effort.

<br>

Plain manifests only describe the desired state and know nothing about how your app actually runs, so backups, failover, and safe upgrades are left to you. An operator is a Custom Resource plus a controller that bakes that operational knowledge into code and automates all of it. The trade off: operators give much more control and automation but cost effort to build and maintain, so they are worth it for complex stateful apps like databases, while plain manifests are enough for simple or stateless ones.

<br>
<br>

Assess the developer experience of plain Kubernetes (kubectl + manifests) versus OpenShift's Source-toImage (S2I) and developer console. What does each optimize for?

<br>

Plain Kubernetes means kubectl and hand written YAML, which gives full control and portability but a steep learning curve. OpenShift's Source to Image builds the image straight from your source repo with no Dockerfile, and its console lets you deploy through a GUI. So Kubernetes optimizes for control and portability, OpenShift optimizes for developer speed and ease.

<br>
<br>

Critically evaluate migrating a workload from OpenShift to Kubernetes.

<br>

Standard objects like Deployments, Services, and Secrets move across cleanly since OpenShift is Kubernetes underneath. The work is in the OpenShift specific parts: Routes become Ingress, BuildConfigs and S2I get replaced by an external CI pipeline, and the random non root UID behavior from SCC has to be handled with PodSecurity and securityContext. It is doable and reduces vendor lock in, but you lose the built in tooling like the console, registry, and monitoring and have to rebuild it yourself.

<br>
<br>
