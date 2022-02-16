# WSL2 debian 11 (Bullseye) setup on Windows 11 for development

## Motivation

I had a Windows 10 setup with WSL running for almost a year. Most of the development was directly on Windows while some was on Linux with Docker desktop on Windows. All was working fine till one day Windows crashed. I had system backups configured with assumption that whole setup can be restored in case needed. I think in general it was ok, but I happened to be in that small percentages where none of the options worked from Windows restore and even the custom backup tool that I was using.

I had to start from scratch installation of Windows 10. This gave me an opportunity to rethink my development setup. I do not know if it is correct or not, but my current idea is now to have below setup.

1. Windows 11 main OS
2. WSL 2 for development. For whatever reason I selected Debian, even though Ubuntu seems to be dominating.
   1. Git on WSL distro as I do not want to have development on Windows
   2. GCM on Windows to share credentials among different distros.
3. VS Code as IDE
   1. Remote WSL
   2. Remote Containers
   3. VS Code server in WSL distro

Above is the main development Debian(11) image. Idea is to keep it as an exported tar file so it can be easily imported in any WSL 2 environment to start development with minimum effort. This base image is then used to develop further images. For my purpose, these are

1. Docker installed and configured for Remote container development
2. Kubernetes cluster environment setup using below flavours. I have no idea at the moment, why one should prefer one over the other. All the tools are changing rapidly. Even [some of the comparisons](https://thechief.io/c/editorial/k3d-vs-k3s-vs-kind-vs-microk8s-vs-minikube/) are not valid in a year from the writing.
   1. [MiniKube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
   2. [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/) needs docker as well
   3. [Mikrok8s](https://microk8s.io/?ref=thechiefio) needs snap tool on Linux
   4. [K3s](https://rancher.com/docs/k3s/latest/en/)
   5. [RKE](https://rancher.com/docs/rke/latest/en/)
3. Apache Airflow image. Any one which works. [Official documentation](https://airflow.apache.org/docs/apache-airflow/stable/installation/index.html#using-managed-airflow-services) provides atleast 4 ways to install Airflow. In addition there are solutions from 3rd party and cloud service providers. I hope to get [Kubernetes option using Kind](https://airflow.apache.org/docs/helm-chart/stable/index.html#) working using Helm Chart. This will also help me (I hope) to learn Docker, Kubernetes on the way.

This repository is simply documentation of what I did to setup my WSL2 development environment using Debian and VS Code.
One can also consider it a customized version of MS official documentation i.e. "[Setup WSL development environment](https://docs.microsoft.com/en-us/windows/wsl/setup/environment#set-up-your-linux-username-and-password)"

## Pre-requisites

### Windows Terminal

Windows terminal is useful tool to have multiple shells from windows and WSL images running in parallel.
In addition it also works great for shell visual configuration using tools like ohmyPosh, ZSH, powerline etc.

[This is a nice collection of Windows Terminal Themes to select, preview and download.](https://windowsterminalthemes.dev/?theme=Blazer)

I use Blazer as my default Theme.

### WSL2 installed and configured

In latest version of Windows 10 and 11 there is 1 command install availabe from WSL2.
In adminstrator mode in cmd or powershell write

 ```bash
 wsl --install Debian
 ```

 Make sure you have WSL version 2 set as the default version.

 ```bash
 wsl --set-default-version 2
 ```

By default Ubuntu will be installed.

Below images are created during this process. I hope to add to the list as I keep working on it. Please clik the link to read details on the images.

NOTE: AS I worked on these images, I found I could compress these exported tar files. Compression would reduce the file size by 1/3! Example command is

 ```powershell
   wsl --export deb11 .\wsl_backup\deb11.tar | tar -czf .\wsl_backup\deb11.tar.gz .\wsl_backup\deb11.tar
   
   rm .\wsl_backup\deb11.tar
   ```

* ## [Base Debian development image with VS Code](base_image_Readme.md)

* ## [Remote Development WSL image based on Docker and Docker Compose V2](RemoteDev_docker_compose_image_Readme.md)

* ## [Kubernetes image based on Kind](Kind_k8s_Readme.md)
* ## [Kubernetes image based on minikube](minikube_k8s_Readme.md)