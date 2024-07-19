# azp-agent

## Docker In Docker (DIND)

Because runing `docker build` in a container needs root priviledge of the host machine in order to use docker engine, most people will not recommand this method.

Especially dockershim (dockerd) was replaced by CRI (containerd) in 2023，trying to use `docker build` in K8s pod is not straightforward anymore.

## Azure Pipelines Agent in AWS EKS

Luckily AWS creates a symbolic link from dockershim.sock to /run/containerd/containerd.sock, enabling DIND to continue functioning seamlessly in the absence of dockerd.

This page will show a way to manage Azure Pipelines Agent in AWS EKS, but you need to understand below three references first.
- [All you need to know about moving to containerd on Amazon EKS](https://aws.amazon.com/tw/blogs/containers/all-you-need-to-know-about-moving-to-containerd-on-amazon-eks/)
- [Demystifying DIND (Docker in Docker) in AWS EKS 1.24+: Adapting to Changes and Innovations](https://medium.com/@iamomerd/demystifying-dind-docker-in-docker-in-aws-eks-1-24-adapting-to-changes-and-innovations-55bea0dc6a01)
- [Run a self-hosted agent in Docker](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops)

## Running Agent as container in VM
- Write a `Dockerfile` to create an image.
- Setup Azure DevOps.

```
docker run -v /var/run/docker.sock:/var/run/docker.sock \
   -e AZP_URL="https://dev.azure.com/xxxxxxxxxx" \
   -e AZP_TOKEN="xxxxxxxxxx" \
   -e AZP_POOL="xxxxxxxxxx" \
   -e AZP_AGENT_NAME="Docker Agent - Linux" \
   --name "azp-agent-linux" \
   azp-agent:v0.1.0
```
