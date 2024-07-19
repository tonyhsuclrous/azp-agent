# azp-agent

## Docker In Docker (DIND)

Because runing `docker build` in a container needs root priviledge of the host machine in order to use docker engine, most people will not recommand this method.

Especially dockershim (dockerd) was replaced by CRI (containerd) in 2023, trying to use `docker build` in K8s pod is not straightforward anymore.

## Azure Pipelines Agent in AWS EKS

Luckily AWS creates a symbolic link from `dockershim.sock` to `/run/containerd/containerd.sock`, enabling DIND to continue functioning seamlessly in the absence of dockerd.

This page will show a way to manage Azure Pipelines Agent in AWS EKS, but you need to understand below three references first.
- [All you need to know about moving to containerd on Amazon EKS](https://aws.amazon.com/tw/blogs/containers/all-you-need-to-know-about-moving-to-containerd-on-amazon-eks/)
- [Demystifying DIND (Docker in Docker) in AWS EKS 1.24+: Adapting to Changes and Innovations](https://medium.com/@iamomerd/demystifying-dind-docker-in-docker-in-aws-eks-1-24-adapting-to-changes-and-innovations-55bea0dc6a01)
- [Run a self-hosted agent in Docker](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops)

## Running Agent as container in VM
- Write a `Dockerfile` to create an image `azp-agent:v0.1.0`.
- Setup Azure DevOps.

```
FROM ubuntu:22.04
ENV TARGETARCH="linux-x64"
# Also can be "linux-arm", "linux-arm64".

RUN apt update
RUN apt upgrade -y
RUN apt install -y curl git jq libicu70
RUN apt install -y ca-certificates
RUN install -m 0755 -d /etc/apt/keyrings
RUN curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
RUN chmod a+r /etc/apt/keyrings/docker.asc
RUN echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
RUN apt update
RUN apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

WORKDIR /azp/

COPY ./start.sh ./
RUN chmod +x ./start.sh

# Create agent user and set up home directory
#RUN useradd -m -d /home/agent agent
#RUN chown -R agent:agent /azp /home/agent

#USER agent
# Another option is to run the agent as root.
ENV AGENT_ALLOW_RUNASROOT="true"

ENTRYPOINT [ "./start.sh" ]
```

```
docker run -v /var/run/docker.sock:/var/run/docker.sock \
   -e AZP_URL="https://dev.azure.com/xxxxxxxxxx" \
   -e AZP_TOKEN="xxxxxxxxxx" \
   -e AZP_POOL="xxxxxxxxxx" \
   -e AZP_AGENT_NAME="Docker Agent - Linux" \
   --name "azp-agent-linux" \
   azp-agent:v0.1.0
```

## Running Agent in EKS
- Create a secret for Azure DevOps.
- Create a deployment with two containers, one for DIND, one for Agent.

```
apiVersion: v1
kind: Secret
metadata:
  name: azdevops
  namespace: azp-agent-linux
type: Opaque
stringData:
  AZP_POOL: "xxxxxxxxxx"
  AZP_TOKEN: "xxxxxxxxxx"
  AZP_URL: "https://dev.azure.com/xxxxxxxxxx"
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azp-agent-linux
  namespace: azp-agent-linux
  labels:
    app: azp-agent-linux
spec:
  selector:
    matchLabels:
      app: azp-agent-linux
  template:
    metadata:
      labels:
        app: azp-agent-linux
    spec:
      containers:
      - name: azp-agent-linux
        image: azp-agent:v0.1.0
        env:
          - name: DOCKER_TLS_CERTDIR
            value: ""
          - name: DOCKER_HOST
            value: tcp://localhost:2375
          - name: AZP_URL
            valueFrom:
              secretKeyRef:
                name: azdevops
                key: AZP_URL
          - name: AZP_TOKEN
            valueFrom:
              secretKeyRef:
                name: azdevops
                key: AZP_TOKEN
          - name: AZP_POOL
            valueFrom:
              secretKeyRef:
                name: azdevops
                key: AZP_POOL
      - name: dind
        image: registry.hub.docker.com/library/docker:dind
        command:
          - sh
        args:
          - /usr/local/bin/dockerd-entrypoint.sh
        env:
          - name: DOCKER_TLS_CERTDIR
            value: ""
          - name: DOCKER_HOST
            value: tcp://localhost:2375
        resources:
          requests:
            memory: "200Mi"
            cpu: "200m"
        securityContext:
          privileged: true
          runAsUser: 0
```
