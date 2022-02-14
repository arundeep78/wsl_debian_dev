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

![Kind create cluster with specific node image works](images/k8s/minikube/minikube_cluster_starts_success.drawio.svg)

If you have installed VS Code extensions above, then details of cluster would be visible in VS Code as well.

- In Kubernetes

![minikube cluster info in vscode kubernetes extension](images/k8s/minikube/minikube_cluster_info_vscode.drawio.svg)

- In Docker extension, as K8s is running in Docker after all
![minikube k8s cluster info in Docker extension in Vscode](images/k8s/minikube/minikube_cluster_info_docker.drawio.svg)

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
