# WSL2 image with Minikube installation

This image is based on [WSL2 docker image](RemoteDev_docker_compose_image_Readme.md). Pleae make sure you have that image or have followed the process to create that image. All the steps in this documentation assumes that all packages and softwares are installed as in docker WSL2 image.

## Motivation

To have a WSL image that can create Kubernetes (K8s) setups as required by certain applications or a custom setup. There are different versions available that help setup Kubernetes. For this image we use [MiniKube](https://minikube.sigs.k8s.io/docs/start/).

## Why MiniKube?

Frankly, there is no preference, as I am not aware of all the nitty-gritties of kubernetes and various flovours of it. I selected minikube as it seems on of the default version for development.I also hope that if I face issues, minikube may have a more active and wide community support. This can then be used as a base image for other applications, that may have similar setups based on minikube.


## Create a new WSL environment

I use docker image as a starting point as Docker seems to be the most popular containerization platform. Minikube has support for other container and VM manager as well, but then it would need another component to learn and configure. Execute below commands to create starting WSL image to develop minikube WSL2 image

```powershell
wsl --import deb11mk .\wsl\deb11mk\ .\wsl_backup\deb11_docker.tar
```

One must change name of images and paths based on the specific setup.

## Install minikube

```zsh
  # Find latest minikube version or set a version that you want to install
  MK_VERSION=$(curl -fsSL -o /dev/null -w "%{url_effective}" https://github.com/kubernetes/minikube/releases/latest | xargs basename)

  # FIND machine's CPU architecture. Currenttly Kind supports ARM and AMD
  ARCHITECTURE=$(dpkg --print-architecture)

  # Download minkube on the system 

  MK=minikube-linux-${ARCHITECTURE}

  curl -LO https://storage.googleapis.com/minikube/releases/${MK_VERSION}/${MK}

  
  # install minikube executable
  sudo install ${MK} /usr/local/bin/minikube
```

## Install kubectl

There are [different ways to install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/). Below is the procedure using binaries

**NOTE**: For some reason `curl -L -s` did not work for me with `dl.k8s.io` for K8S_VERSION, so I changed it to direct URL

```zsh
# kubectl version

K8S_VERSION=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)

# Download Kubectl
curl -O "https://dl.k8s.io/release/${K8S_VERSION}/bin/linux/${ARCHITECTURE}/kubectl"

# Install Kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

```

## Install Helm

[Install Helm](https://helm.sh/docs/intro/install/) to create and manage Kubernetes deployments.

```zsh
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh
./get_helm.sh
```

## Configure autocompletion

In this image we have ZSH configured as default shell using Oh My Posh. Execute below commands to add autocompletion for Kind and kubectl for z sh.


**kubectl,minikube and helm**: Add plugins to `.zshrc` file for Oh My ZSH or execute similar commands as above, but for kubectl and helm

```zsh
plugins =(
        #other plugins
        kubectl
        helm
        minikube
)
```

## Install VS Code extensions

Install below extesions inside WSL VS Code server.

- Install [Kubernetes extension](https://marketplace.visualstudio.com/items?itemName=ms-kubernetes-tools.vscode-kubernetes-tools) in VS Code.
- Install [Helm Intellisense](https://marketplace.visualstudio.com/items?itemName=Tim-Koehler.helm-intellisense)


### Create a basic cluster

```zsh
minikube start
```


**It works!!!!** :happy:

![minikube create cluster with specific node image works](images/k8s/minikube/minikube_cluster_starts_success.drawio.svg)

If you have installed VS Code extensions above, then details of cluster would be visible in VS Code as well.

- In Kubernetes

![minikube cluster info in vscode kubernetes extension](images/k8s/minikube/minikube_cluster_info_vscode.drawio.svg)

- In Docker extension, as K8s is running in Docker after all
  
![minikube k8s cluster info in Docker extension in Vscode](images/k8s/minikube/minikube_cluster_info_docker.drawio.svg)

**Problem**: "This container is having issues accessing https://k8s.gcr.io"

In the oput above of `minikube start` you this message. However, when I login in to container, I can reach to the repository. However, I cannot reach to registry.docker.io or registry-1.docker.io

```zsh
docker exec -it minikube bash
root@minikube:/# ping k8s.gcr.io
PING googlecode.l.googleusercontent.com (74.125.133.82) 56(84) bytes of data.
64 bytes from wo-in-f82.1e100.net (74.125.133.82): icmp_seq=1 ttl=105 time=42.3 ms
root@minikube:/# ping docker.com
PING docker.com (54.198.192.151) 56(84) bytes of data.

root@minikube:/# ping docker.io
PING docker.io (44.197.137.126) 56(84) bytes of data.

root@minikube:/# traceroute registry-1.docker.io
registry-1.docker.io: No address associated with hostname
Cannot handle "host" cmdline arg `registry-1.docker.io` on position 1 (argc 1)

root@minikube:/# ping registry.docker.io
ping: registry.docker.io: Name or service not known

```

I logged out and logged in again to the minikube docker container. Now registry-1.docker.io works!! However, registry.docker.io still does not work

After yet another logout-login step it stopped working again. Not sure where is the issue.

```zsh
root@minikube:/# ping registry.docker.com
PING registry.docker.com (205.251.197.9) 56(84) bytes of data.
64 bytes from ns-1289.awsdns-33.org (205.251.197.9): icmp_seq=1 ttl=241 time=24.4 ms
64 bytes from ns-1289.awsdns-33.org (205.251.197.9): icmp_seq=2 ttl=241 time=24.5 ms

root@minikube:/# ping registry-1.docker.io
PING registry-1.docker.io (174.129.220.74) 56(84) bytes of data.

root@minikube:/# ping registry.docker.io
ping: registry.docker.io: Name or service not known
```

I stopped minikube cluster, purged the data/cache and started the cluster again. This time I did not get a warning about `k8s.gcr.io`. But still it cannot reach registry.docker.io!

```zsh
 minikube start
😄  minikube v1.25.1 on Debian 11.2 (amd64)
✨  Automatically selected the docker driver
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🔥  Creating docker container (CPUs=2, Memory=2200MB) ...
🐳  Preparing Kubernetes v1.23.1 on Docker 20.10.12 ...
    ▪ kubelet.housekeeping-interval=5m
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

```zsh
root@minikube:/# traceroute k8s.gcr.io
traceroute to k8s.gcr.io (142.250.13.82), 30 hops max, 60 byte packets
 1  host.minikube.internal (192.168.49.1)  0.032 ms  0.006 ms  0.004 ms

root@minikube:/# traceroute registry.docker.io
registry.docker.io: Temporary failure in name resolution
Cannot handle "host" cmdline arg `registry.docker.io` on position 1 (argc 1)

root@minikube:/# traceroute registry-1.docker.io
registry-1.docker.io: No address associated with hostname
Cannot handle "host" cmdline arg `registry-1.docker.io` on position 1 (argc 1)
```


**Memory usage**: heavy

On my machine this basic cluster with 1 node and no actual pod running took about 7GB on my WSL2 VM. I do have another WSL2 image running with VS code, where this repository is being updated. At the same time minikube WSL2 image is also running with VS Code to test extensions and work on config.yaml etc.

![WSL VM memory usage after running minikube blank cluster](images/k8s/minikube/minikube_wsl_vm_memory_usage_blank.drawio.svg)

Inside minikube WSL

![Minikube memory usage inside WSL](images/k8s/minikube/minikube_mem_usage_inside_wsl.drawio.svg)

## Export the minikube image as WSL image

Now we have a basic minikube distribution/flavour of K8s running on this image. We can export this image and save it for later use as starting point.


1. Delete the temporary cluster

   ```zsh
   minikube stop
   ```

2. Remove minikube container and all details
   
   ```zsh
   minikube delete --purge
   ```
   
3. Remove any left over docker images

   ```zsh
   docker image prune
   ```

4. Export the image as mk WSL image

   ```powershell
   wsl --export deb11kindk8s .\wsl_backup\deb11_kind.tar | gzip.exe -9 .\wsl_backup\deb11_kind.tar
   ```

   Note that [gzip](http://gnuwin32.sourceforge.net/packages/gzip.htm) is piped to compress the tar file. Later I found that Windows has inbuild `tar` and it worked faster than gzip.

   ```powershell
   wsl --export deb11kindk8s .\wsl_backup\deb11_kind.tar | tar -czf .\wsl_backup\deb11_kind.tar.gz .\wsl_backup\deb11_kind.tar
   
   rm .\wsl_backup\deb11_kind.tar
   ```

   However, I don't know how to improve this command to avoid repeating filenames.

NOTE: Compressed image file is now about 600 MB.
