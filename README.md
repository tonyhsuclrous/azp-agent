# azp-agent

## Docker In Docker (DIND)

Because runing `docker build` in a container needs root priviledge of the host machine in order to use docker engine, most people will not recommand this method.

Especially dockershim (dockerd) was replaced by CRI (containerd) in 2023，trying to use `docker build` in K8s pod is not straightforward anymore.

## Azure Pipelines Agent in AWS EKS

Luckily AWS creates a symbolic link from dockershim.sock to /run/containerd/containerd.sock, enabling DIND to continue functioning seamlessly in the absence of dockerd.
