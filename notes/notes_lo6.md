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

Evaluate running CI/CD build agents (Jenkins agents, GitLab runners, Tekton) inside Kubernetes versus on
dedicated VMs, considering isolation, autoscaling, and noisy-neighbor effects.

<br>

On Kubernetes each build runs in its own pod, so you get clean isolation and autoscaling: pods spin up when jobs arrive and disappear when done, so you only use resources while building. The catch is the noisy neighbor effect, since heavy builds share nodes and can starve each other unless you set resource requests and limits. Dedicated VMs give stronger, more predictable isolation but they sit idle and cost money between builds and do not scale automatically. So Kubernetes wins on cost and elasticity, VMs win on predictable isolation.

<br>
<br>

How does Kubernetes integrate with cloud environments and dynamic capacity provisioning?

<br>

Kubernetes plugs into clouds through provider integrations, so a LoadBalancer Service creates a real cloud load balancer and a StorageClass provisions real cloud disks through CSI. For capacity, the Cluster Autoscaler adds or removes nodes as pods need them, while the Horizontal Pod Autoscaler adds or removes pod replicas based on load. Together that means the cluster grows and shrinks with demand instead of you provisioning servers by hand.

<br>
<br>

Critically evaluate the default security and hardening of OpenShift versus vanilla Kubernetes. Why do
enterprises in regulated industries often choose OpenShift?

<br>

Vanilla Kubernetes is permissive out of the box: it will happily run containers as root and leaves most hardening to you. OpenShift ships locked down by default, most importantly through Security Context Constraints that block root and force containers to run as a random non root user, plus stricter defaults and an integrated authentication stack. Regulated industries like banking and healthcare choose OpenShift because those secure defaults, plus vendor support and easier compliance and auditing from Red Hat, save them from hardening everything themselves.

<br>
<br>

Critically evaluate the learning curve and required skill set of OpenShift versus plain Kubernetes. Does
OpenShift's added abstraction help or hinder a team new to containers?

<br>

Plain Kubernetes has a steep curve because you must learn manifests, image building, and networking before anything ships. OpenShift adds friendly abstractions like Source to Image and a web console that let a new team deploy quickly without writing YAML or Dockerfiles. So for a team new to containers the abstraction mostly helps them get productive fast, the downside being that it hides how things really work and adds its own OpenShift specific concepts to learn on top of Kubernetes.

<br>
<br>

Compare the resource footprint of full Kubernetes versus k3s for a small on-premise deployment. When is
full Kubernetes overkill?

<br>

k3s is a lightweight Kubernetes: a single small binary with a light database and reduced memory use, so it runs comfortably on small servers or edge devices. Full Kubernetes has a heavier control plane and normally wants multiple nodes and more resources and operational effort. For a small on premise setup full Kubernetes is overkill when you have only a handful of nodes and no need for high availability or huge scale, and k3s gives you the same API with far less overhead.

<br>
<br>
