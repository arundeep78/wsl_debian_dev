# WSL2 image with Kind installation

This image is based on [WSL2 docker image](RemoteDev_docker_compose_image_Readme.md). Pleae make sure you have that image or have followed the process to create that image. All the steps in this documentation assumes that all packages and softwares are installed as in docker WSL2 image.

## Motivation

To have a WSL image that can create Kubernetes (K8s) setups as required by certain applications or a custom setup. There are different versions available that help setup Kubernetes. For this image we use [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

## Why Kind?

Frankly, there is no preference, as I am not aware of all the nitty-gritties of kubernetes and various flovours of it. I selected Kind because [Apache Airflow sample K8s setup documentation is based on Kind](https://airflow.apache.org/docs/helm-chart/stable/quick-start.html). Rather than trying to have K8s setup with other flavours, I preferred to follow the steps and have Kind installed as a base. This can then be used as a base image for other applications, that may have similar setups based on Kind.

**NOTE:**

1. Kind currently seems to have [issue with SNAP package manager for Linux](https://kind.sigs.k8s.io/docs/user/known-issues/#docker-installed-with-snap). Particularly if Docker on WSL is installed using SNAP package manager, it may have issues. There is a workaround available. But, in this Docker WSL image is not based on SNAP, so we are good.
2. [WSL2 and K8s services with session affinity](https://kind.sigs.k8s.io/docs/user/using-wsl2/#kubernetes-service-with-session-affinity): There is an issue at the time for writing due to a missing module in WSL2 kernel. There is a procedure given on the link to update the WSL2 kernel to fix the issue. I am not sure, If I need that specific configuration yet, so I continue with default WSL2 kernel.
3. Kind currently seems to support Docker as CRI (Container Runtime Interface) and assumes it is installed. One of the reason to use Debian docker WSL image as base.

## Create a new WSL environment

I use docker image as a starting point as Kind seems to have docker as pre-requisite for its operations. Also, if you want to deploy your own images to Kind, you might need docker installed on the system. Execute below commands in cmd or powershell to import the Docker WSL image created earlier.

```powershell
wsl --import deb11kindk8s .\wsl\deb11kindk8s\ .\wsl_backup\deb11_docker.tar
```

One must change name of images and paths based on the specific setup.

## Install kind

```zsh
  # Find latest kind version
  KIND_VERSION=$(curl -fsSL -o /dev/null -w "%{url_effective}" https://github.com/kubernetes-sigs/kind/releases/latest | xargs basename)

  # FIND machine's CPU architecture. Currenttly Kind supports ARM and AMD
  ARCHITECTURE=$(dpkg --print-architecture)

  # Download kind on the system somewhere in PATH

  sudo curl -Lo /usr/local/bin/kind https://kind.sigs.k8s.io/dl/${KIND_VERSION}/kind-linux-${ARCHITECTURE}

  # make kind executable
  sudo chmod +x /usr/local/bin/kind
```

## Install kubectl

There are [different ways to install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/). Below is the procedure using binaries

```zsh
# Doanload Kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

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

**kind**: execute below commands

```zsh
kind completion zsh > _kind
sudo mv _kind > /usr/share/zsh/vendor-completions/_kind
```

**kubectl and helm**: Add plugins to `.zshrc` file for Oh My ZSH or execute similar commands as above, but for kubectl and helm

```zsh
plugins =(
        #other plugins
        kubectl
        helm
)
```

## Install VS Code extensions

Install below extesions inside WSL VS Code server.

- Install [Kubernetes extension](https://marketplace.visualstudio.com/items?itemName=ms-kubernetes-tools.vscode-kubernetes-tools) in VS Code.
- Install [Kubernetes Kind](https://marketplace.visualstudio.com/items?itemName=ms-kubernetes-tools.kind-vscode) extension on VS Code.

## Test kind

### Create a basic cluster

```zsh
kind create cluster --name testcluster
```

**It failed! :weary:**

![Kind create cluster command failed](images/k8s/kind/kind_create_cmd_fail.drawio.svg
)

#### Error message

 seems to be related to failure to bring up control-plane.

```zsh
I0114 17:17:05.425093     218 round_trippers.go:454] GET https://testcluster-control-plane:6443/healthz?timeout=10s  in 0 milliseconds
I0114 17:17:05.925511     218 round_trippers.go:454] GET https://testcluster-control-plane:6443/healthz?timeout=10s  in 0 milliseconds
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.

        Unfortunately, an error has occurred:
                timed out waiting for the condition

        This error is likely caused by:
                - The kubelet is not running
                - The kubelet is unhealthy due to a misconfiguration of the node in some way (required cgroups disabled)

        If you are on a systemd-powered system, you can try to troubleshoot the error with the following commands:
                - 'systemctl status kubelet'
                - 'journalctl -xeu kubelet'

        Additionally, a control plane component may have crashed or exited when started by the container runtime.
        To troubleshoot, list all containers using your preferred container runtimes CLI.
```

#### Reason?

I don't know, but I suspect this has something to do with `/healthz`. Which means someone had a different keyboard layout.

How about starting a kubernetes cluster is as easy as running `kind create cluster`

#### Solution

Create a cluster config.yaml file with different image version. Each [kind version/release has a list of image versions](https://github.com/kubernetes-sigs/kind/releases) that can be used. So, make sure to pick the one from the list.

In my case it was using default latest version i.e. 1.2.1.

It worked by changing the image version to 1.2.0 for both worker and control-plane nodes. Config file is as simple as

```zsh
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: kind-test
nodes: 
- role: control-plane
  image: kindest/node:v1.20.7@sha256:cbeaf907fc78ac97ce7b625e4bf0de16e3ea725daf6b04f930bd14c67c671ff9
- role: worker
  image: kindest/node:v1.20.7@sha256:cbeaf907fc78ac97ce7b625e4bf0de16e3ea725daf6b04f930bd14c67c671ff9
```

**It works!!!!** :happy:

![Kind create cluster with specific node image works](images/k8s/kind/kind_cluster_works_log.drawio.svg)

If you have installed VS Code extensions above, then details od cluster would be visible in VS Code as well.

- In Kubernetes
![kind cluster info in vscode kubernetes extension](images/k8s/kind/kind_cluster_info_vscode.drawio.svg)

- In Docker extension, as K8s is running in Docker after all
![Kind k8s cluster info in Docker extension in Vscode](images/k8s/kind/kind_cluster_info_docker.drawio.svg)

**Memory usage**: heavy

On my machine this basic cluster with 2 nodes and no actual pod running took about 7GB on my WSL2 VM. I do have another WSL2 image running with VS code, where this repository is being updated. At the same time Kind WSL2 image is also running with VS Code to test extensions and work on config.yaml etc.

![WSL VM memory usage after running Kind blank cluster](images/k8s/kind/kind_wsl_vm_memory_usage_blank.drawio.svg)

Inside Kind WSL

![Kind memory usage inside WSL](images/k8s/kind/kind_mem_usage_inside_wsl.drawio.svg)

## Export the Kind image as WSL image

Now we have a basic kind distribution/flavour of K8s running on this image. We can exxport this image and save it for later use as starting point.

This version atleast as an issue with v1.21 image. So it has a basic config file stating which node version is working for this kind release.

If we update kind then we might have to update node image version as well.

1. Delete the temporary cluster

   ```zsh
   kind delete cluster --name kind-test
   ```

2. Remove any left over docker images

   ```zsh
   docker image prune
   ```

3. Export the image as kindk8s WSL image

   ```powershell
   wsl --export deb11kindk8s .\wsl_backup\deb11_kind.tar | gzip.exe -9 .\wsl_backup\deb11_kind.tar
   ```

   Note that [gzip](http://gnuwin32.sourceforge.net/packages/gzip.htm) is piped to compress the tar file. Later I found that Windows has inbuild `tar` and it worked faster than gzip.

   ```powershell
   wsl --export deb11kindk8s .\wsl_backup\deb11_kind.tar | tar -czf .\wsl_backup\deb11_kind.tar.gz .\wsl_backup\deb11_kind.tar
   
   rm .\wsl_backup\deb11_kind.tar
   ```

   However, I don't know how to improve this command to avoid repeating filenames.

NOTE: Compresses image file is now about 700 MB.
