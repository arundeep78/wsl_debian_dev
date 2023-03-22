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
# Download Kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install Kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

```

## Install krew

[krew](https://krew.sigs.k8s.io/docs/user-guide/setup/install/) is a plugin manager for `kubectl`. It helps to add usefull additional functionality to kubectl.

Follow steps on the ofifical website. 

1. Make sure `git` is installed.
2. Execute below commands in for `zsh` or `bash`

   ```bash
      (
         set -x; cd "$(mktemp -d)" &&
         OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
         ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
         KREW="krew-${OS}_${ARCH}" &&
         curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
         tar zxvf "${KREW}.tar.gz" &&
         ./"${KREW}" install krew
      )
   ```

3. Add the $HOME/.krew/bin directory to your PATH environment variable. To do this, update your .bashrc or .zshrc file and append the following line:

   ```bash
      export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
   ```

## Install Helm

[Install Helm](https://helm.sh/docs/intro/install/) to create and manage Kubernetes deployments.

```zsh
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh
./get_helm.sh
```

## Install cmctl

This tool is required for [cert-manager](https://cert-manager.io/docs/usage/cmctl/) to issue HTTPS certificates for DNS names.

```zsh
ARCHITECTURE=$(dpkg --print-architecture)

CMCTL_VERSION=$(curl -fsSL -o /dev/null -w "%{url_effective}" https://github.com/cert-man
ager/cert-manager/releases/latest | xargs basename)

curl -sSLO https://github.com/cert-manager/cert-manager/releases/download/$CMCTL_VERSION/
cmctl-linux-$ARCHITECTURE.tar.gz

tar xzf cmctl-linux-$ARCHITECTURE.tar.gz

sudo mv cmctl /usr/local/bin


```

## Install k9s

[k9s is an amzing friendly tool](https://github.com/derailed/k9s) to work with k8s cluster. It provides an interactive interface to work with cluster and make it a breeze in comparison to kubectl.

```zsh
curl -sS https://webinstall.dev/k9s | bash
```

## Configure autocompletion

In this image we have ZSH configured as default shell using Oh My Posh. Execute below commands to add autocompletion for Kind and kubectl for z sh.

**kind**: execute below commands

```zsh
kind completion zsh > _kind
sudo mv _kind /usr/share/zsh/vendor-completions/_kind
```

**cmctl** : execute below commands

```zsh
cmctl completion zsh > _cmctl
sudo mv _cmctl /usr/share/zsh/vendor-completions/_cmctl
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

## Check internet access from nodes

This section is updated after trying to install airflow and having issues with downloading docker repositories. I must say that many of the details below does not mean much to me, as I do not understand all of that information. But, it might help someone else to figure it out the issue.

Error message received while installing airflow was

```zsh
"Failed to pull image "redis:6-buster": rpc error: code = Unknown desc = failed to pull and unpack image "docker.io/library/redis:6-buster": failed to resolve reference "docker.io/library/redis:6-buster": failed to do request: Head "https://registry-1.docker.io/v2/library/redis/manifests/6-buster": dial tcp: lookup registry-1.docker.io on 172.19.0.1:53: no such host"
```

<details>

<summary>
Expand for details
</summary>

## WSL HOST information

### Check ip information

```zsh
$hostname -I
172.25.120.157 172.18.0.1 172.17.0.1 172.19.0.1 fc00:f853:ccd:e793::1

$ip route list
default via 172.25.112.1  dev eth0
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
172.18.0.0/16 dev br-15020f7aae7f proto kernel scope link src 172.18.0.1 linkdown
172.19.0.0/16 dev br-cfec1d7dd4c8 proto kernel scope link src 172.19.0.1 linkdown
172.25.112.0/20 dev eth0 proto kernel scope link src 172.25.120.157

$ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: bond0: <BROADCAST,MULTICAST,MASTER> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether fa:cd:08:d8:b2:89 brd ff:ff:ff:ff:ff:ff
3: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 62:aa:45:82:eb:0f brd ff:ff:ff:ff:ff:ff
4: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
5: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/sit 0.0.0.0 brd 0.0.0.0
6: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:15:5d:20:05:5d brd ff:ff:ff:ff:ff:ff
    inet 172.25.120.157/20 brd 172.25.127.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::215:5dff:fe20:55d/64 scope link
       valid_lft forever preferred_lft forever
7: br-15020f7aae7f: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:88:ff:bb:39 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-15020f7aae7f
       valid_lft forever preferred_lft forever
8: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:6d:82:bc:61 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:6dff:fe82:bc61/64 scope link
       valid_lft forever preferred_lft forever
9: br-cfec1d7dd4c8: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:d4:ff:a2:79 brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.1/16 brd 172.19.255.255 scope global br-cfec1d7dd4c8
       valid_lft forever preferred_lft forever
    inet6 fc00:f853:ccd:e793::1/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::42:d4ff:feff:a279/64 scope link
       valid_lft forever preferred_lft forever
    inet6 fe80::1/64 scope link
       valid_lft forever preferred_lft forever
```

### Check DNS server information

```zsh
$cat /etc/nsswitch.conf
hosts:          files dns
networks:       files

protocols:      db files
services:       db files
ethers:         db files
rpc:            db files

netgroup:       nis

cat /etc/resolv.conf
# This file was automatically generated by WSL. To stop automatic generation of this file, add the following entry to /etc/wsl.conf:
# [network]
# generateResolvConf = false
nameserver 172.25.112.1 # IP address of the WSL interface in windows
```

### Check docker daemon settings

```zsh
 cat /etc/docker/daemon.json
{
        "features" : {
                "buildkit": true
        }
}
```

### access to standard internet address

```zsh
$ping google.com
PING google.com (216.239.32.10) 56(84) bytes of data.
64 bytes from ns1.google.com (216.239.32.10): icmp_seq=1 ttl=106 time=41.0 ms
64 bytes from ns1.google.com (216.239.32.10): icmp_seq=2 ttl=106 time=40.3 ms

$ping yahoo.com
PING yahoo.com (74.6.143.25) 56(84) bytes of data.
64 bytes from media-router-fp73.prod.media.vip.bf1.yahoo.com (74.6.143.25): icmp_seq=1 ttl=48 time=148 ms
64 bytes from media-router-fp73.prod.media.vip.bf1.yahoo.com (74.6.143.25): icmp_seq=2 ttl=46 time=123 ms

$ping m-w.com
PING m-w.com (205.251.196.60) 56(84) bytes of data.
64 bytes from ns-1084.awsdns-07.org (205.251.196.60): icmp_seq=1 ttl=241 time=36.6 ms
64 bytes from ns-1084.awsdns-07.org (205.251.196.60): icmp_seq=2 ttl=241 time=33.5 ms
```

### access to image repositories

WSL host resolves names of these repositories

```zsh
$ping docker.com
PING docker.com (54.198.192.151) 56(84) bytes of data.

$ping registry.docker.com
PING registry.docker.com (3.85.67.18) 56(84) bytes of data.

$ping registry.docker.io
PING registry-1.docker.io (54.198.211.201) 56(84) bytes of data.

$ping registry-1.docker.io
PING registry-1.docker.io (54.198.211.201) 56(84) bytes of data.

$ping k8s.gcr.io
PING googlecode.l.googleusercontent.com (142.250.145.82) 56(84) bytes of data.
64 bytes from eb-in-f82.1e100.net (142.250.145.82): icmp_seq=1 ttl=105 time=45.4 ms
64 bytes from eb-in-f82.1e100.net (142.250.145.82): icmp_seq=2 ttl=105 time=46.7 ms
```

### DNS information on repositories

```zsh
$dig registry.docker.io

 DiG 9.16.22-Debian  registry.docker.io
;; global options: +cmd
;; Got answer:
;; -HEADER- opcode: QUERY, status: NOERROR, id: 57336
;; flags: qr rd ad; QUERY: 1, ANSWER: 16, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;registry.docker.io.            IN      A

;; ANSWER SECTION:
registry.docker.io.     0       IN      CNAME   registry-1.docker.io.
registry-1.docker.io.   0       IN      A       54.86.228.181
registry-1.docker.io.   0       IN      A       54.210.12.153
registry-1.docker.io.   0       IN      A       52.72.186.182
registry-1.docker.io.   0       IN      A       34.203.135.183
registry-1.docker.io.   0       IN      A       52.0.218.102
registry-1.docker.io.   0       IN      A       52.200.37.142
registry-1.docker.io.   0       IN      A       34.200.175.181
registry-1.docker.io.   0       IN      A       52.202.132.224
ns-1168.awsdns-18.org.  0       IN      A       205.251.196.144
ns-1827.awsdns-36.co.uk. 0      IN      A       205.251.199.35
ns-421.awsdns-52.com.   0       IN      A       205.251.193.165
ns-513.awsdns-00.net.   0       IN      A       205.251.194.1
ns-1168.awsdns-18.org.  0       IN      AAAA    2600:9000:5304:9000::1
ns-1827.awsdns-36.co.uk. 0      IN      AAAA    2600:9000:5307:2300::1
ns-421.awsdns-52.com.   0       IN      AAAA    2600:9000:5301:a500::1

;; Query time: 50 msec
;; SERVER: 172.25.112.1#53(172.25.112.1)
;; WHEN: Tue Feb 22 23:53:44 CET 2022
;; MSG SIZE  rcvd: 532


$dig registry-1.docker.io
;  DiG 9.16.22-Debian registry-1.docker.io
;; global options: +cmd
;; Got answer:
;; -HEADER- opcode: QUERY, status: NOERROR, id: 13277
;; flags: qr rd ad; QUERY: 1, ANSWER: 16, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;registry-1.docker.io.          IN      A

;; ANSWER SECTION:
registry-1.docker.io.   0       IN      A       34.237.244.67
registry-1.docker.io.   0       IN      A       52.203.238.92
registry-1.docker.io.   0       IN      A       54.210.12.153
registry-1.docker.io.   0       IN      A       44.196.236.180
registry-1.docker.io.   0       IN      A       52.202.132.224
registry-1.docker.io.   0       IN      A       54.197.112.205
registry-1.docker.io.   0       IN      A       34.200.175.181
registry-1.docker.io.   0       IN      A       54.198.211.201
ns-1168.awsdns-18.org.  0       IN      A       205.251.196.144
ns-1827.awsdns-36.co.uk. 0      IN      A       205.251.199.35
ns-421.awsdns-52.com.   0       IN      A       205.251.193.165
ns-513.awsdns-00.net.   0       IN      A       205.251.194.1
ns-1168.awsdns-18.org.  0       IN      AAAA    2600:9000:5304:9000::1
ns-1827.awsdns-36.co.uk. 0      IN      AAAA    2600:9000:5307:2300::1
ns-421.awsdns-52.com.   0       IN      AAAA    2600:9000:5301:a500::1
ns-513.awsdns-00.net.   0       IN      AAAA    2600:9000:5302:100::1

;; Query time: 29 msec
;; SERVER: 172.25.112.1#53(172.25.112.1)
;; WHEN: Wed Feb 23 12:32:16 CET 2022
;; MSG SIZE  rcvd: 530


$dig k8s.gcr.io

;  DiG 9.16.22-Debian  k8s.gcr.io
;; global options: +cmd
;; Got answer:
;; -HEADER- opcode: QUERY, status: NOERROR, id: 18666
;; flags: qr rd ad; QUERY: 1, ANSWER: 10, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;k8s.gcr.io.                    IN      A

;; ANSWER SECTION:
K8S.GCr.io.             0       IN      CNAME   googlecode.l.googleusercontent.com.
googlecode.l.GOoGLEusErCOntEnt.com. 0 IN A      142.250.145.82
ns2.google.com.         0       IN      A       216.239.34.10
ns1.google.com.         0       IN      A       216.239.32.10
ns3.google.com.         0       IN      A       216.239.36.10
ns4.google.com.         0       IN      A       216.239.38.10
ns2.google.com.         0       IN      AAAA    2001:4860:4802:34::a
ns1.google.com.         0       IN      AAAA    2001:4860:4802:32::a
ns3.google.com.         0       IN      AAAA    2001:4860:4802:36::a
ns4.google.com.         0       IN      AAAA    2001:4860:4802:38::a

;; Query time: 20 msec
;; SERVER: 172.25.112.1#53(172.25.112.1)
;; WHEN: Tue Feb 22 23:57:41 CET 2022
;; MSG SIZE  rcvd: 424
```

## KIND worker node network information

Start kind cluster with node image 1.20. It starts fine without giving any errors or warnings

```zsh
$kind create cluster --config=config.yaml
Creating cluster "kind-test" ...
 ✓ Ensuring node image (kindest/node:v1.20.7) 🖼
 ✓ Preparing nodes 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kind-test"
You can now use your cluster with:

kubectl cluster-info --context kind-kind-test

Thanks for using kind! 😊
```

Login to the worker node

```zsh
$docker exec -it kind-test-worker bash
root@kind-test-worker:/#
```

Install basic tools: Internet works for all these tasks.

```zsh
$apt update

$apt install iputils-ping tcpdump dnsutils traceroute -y
```

### Check ip information-2

```zsh
root@kind-test-worker:/# hostname -I
172.19.0.3 fc00:f853:ccd:e793::3

root@kind-test-worker:/# ip route list
default via 172.19.0.1 dev eth0
10.244.0.0/24 via 172.19.0.2 dev eth0
172.19.0.0/16 dev eth0 proto kernel scope link src 172.19.0.3

root@kind-test-worker:/# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
3: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/sit 0.0.0.0 brd 0.0.0.0
72: eth0@if73: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:13:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.19.0.3/16 brd 172.19.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fc00:f853:ccd:e793::3/64 scope global nodad
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe13:3/64 scope link
       valid_lft forever preferred_lft forever
```

### Check DNS server information-2

```zsh
root@kind-test-worker:/# cat /etc/nsswitch.conf
hosts:          files dns
networks:       files

protocols:      db files
services:       db files
ethers:         db files
rpc:            db files

netgroup:       nis

root@kind-test-worker:/# cat /etc/resolv.conf
nameserver 172.19.0.1
options ndots:0

```

### access to standard internet address-2

```zsh
root@kind-test-worker:/# ping google.com
PING google.com (216.239.38.10) 56(84) bytes of data.
64 bytes from ns4.google.com (216.239.38.10): icmp_seq=1 ttl=105 time=43.6 ms

root@kind-test-worker:/# ping yahoo.com
PING yahoo.com (74.6.231.20) 56(84) bytes of data.
64 bytes from media-router-fp73.prod.media.vip.ne1.yahoo.com (74.6.231.20): icmp_seq=1 ttl=47 time=185 ms

root@kind-test-worker:/# ping m-w.com
PING m-w.com (205.251.196.60) 56(84) bytes of data.
64 bytes from ns-1084.awsdns-07.org (205.251.196.60): icmp_seq=1 ttl=240 time=21.9 ms

```

### access to image repositories-2

It seems that registry.docker.io has problem with name resolution

```zsh
root@kind-test-worker:/# ping docker.com
PING docker.com (54.198.192.151) 56(84) bytes of data.

root@kind-test-worker:/# ping registry.docker.com
PING registry.docker.com (205.251.197.9) 56(84) bytes of data.
64 bytes from ns-1289.awsdns-33.org (205.251.197.9): icmp_seq=1 ttl=241 time=25.1 ms

root@kind-test-worker:/# ping registry.docker.io
ping: registry.docker.io: Temporary failure in name resolution

root@kind-test-worker:/# ping registry-1.docker.io
PING registry-1.docker.io (34.200.175.181) 56(84) bytes of data.

root@kind-test-worker:/# ping k8s.gcr.io
PING googlecode.l.googleusercontent.com (142.251.18.82) 56(84) bytes of data.
64 bytes from er-in-f82.1e100.net (142.251.18.82): icmp_seq=1 ttl=107 time=41.6 ms
```

### DNS information on repositories-2

registry.docker.io gets a DNS information, but is diffrent than the one on the WSL host DNS. Same is the case for k8s.gcr.io

```zsh
root@kind-test-worker:/# dig registry.docker.io

; DiG 9.16.8-Ubuntu  registry.docker.io
;; global options: +cmd
;; Got answer:
;; -HEADER- opcode: QUERY, status: NOERROR, id: 40316
;; flags: qr rd ad; QUERY: 1, ANSWER: 16, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;registry.docker.io.            IN      A

;; ANSWER SECTION:
registry.docker.io.     0       IN      CNAME   registry-1.docker.io.
ReGiSTry-1.docker.io.   0       IN      A       34.196.250.152
ReGiSTry-1.docker.io.   0       IN      A       52.0.218.102
ReGiSTry-1.docker.io.   0       IN      A       54.197.112.205
ReGiSTry-1.docker.io.   0       IN      A       52.202.132.224
ReGiSTry-1.docker.io.   0       IN      A       52.205.127.201
ReGiSTry-1.docker.io.   0       IN      A       52.72.186.182
ReGiSTry-1.docker.io.   0       IN      A       44.196.236.180
ReGiSTry-1.docker.io.   0       IN      A       52.72.252.48
ns-1168.awsdns-18.org.  0       IN      A       205.251.196.144
ns-1827.awsdns-36.co.uk. 0      IN      A       205.251.199.35
ns-421.awsdns-52.com.   0       IN      A       205.251.193.165
ns-513.awsdns-00.net.   0       IN      A       205.251.194.1
ns-1168.awsdns-18.org.  0       IN      AAAA    2600:9000:5304:9000::1
ns-1827.awsdns-36.co.uk. 0      IN      AAAA    2600:9000:5307:2300::1
ns-421.awsdns-52.com.   0       IN      AAAA    2600:9000:5301:a500::1

;; Query time: 29 msec
;; SERVER: 172.19.0.1#53(172.19.0.1)
;; WHEN: Wed Feb 23 11:20:52 UTC 2022
;; MSG SIZE  rcvd: 432

root@kind-test-worker:/# dig registry-1.docker.io

;  DiG 9.16.8-Ubuntu registry-1.docker.io
;; global options: +cmd
;; Got answer:
;; -HEADER- opcode: QUERY, status: NOERROR, id: 60695
;; flags: qr rd ad; QUERY: 1, ANSWER: 16, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;registry-1.docker.io.          IN      A

;; ANSWER SECTION:
registry-1.docker.io.   0       IN      A       54.198.211.201
registry-1.docker.io.   0       IN      A       44.196.236.180
registry-1.docker.io.   0       IN      A       52.202.132.224
registry-1.docker.io.   0       IN      A       52.72.186.182
registry-1.docker.io.   0       IN      A       34.237.244.67
registry-1.docker.io.   0       IN      A       54.86.228.181
registry-1.docker.io.   0       IN      A       52.200.78.26
registry-1.docker.io.   0       IN      A       52.72.252.48
ns-1168.awsdns-18.org.  0       IN      A       205.251.196.144
ns-1827.awsdns-36.co.uk. 0      IN      A       205.251.199.35
ns-421.awsdns-52.com.   0       IN      A       205.251.193.165
ns-513.awsdns-00.net.   0       IN      A       205.251.194.1
ns-1168.awsdns-18.org.  0       IN      AAAA    2600:9000:5304:9000::1
ns-1827.awsdns-36.co.uk. 0      IN      AAAA    2600:9000:5307:2300::1
ns-421.awsdns-52.com.   0       IN      AAAA    2600:9000:5301:a500::1
ns-513.awsdns-00.net.   0       IN      AAAA    2600:9000:5302:100::1

;; Query time: 29 msec
;; SERVER: 172.19.0.1#53(172.19.0.1)
;; WHEN: Wed Feb 23 11:30:02 UTC 2022
;; MSG SIZE  rcvd: 426


root@kind-test-worker:/# dig k8s.gcr.io

;  DiG 9.16.8-Ubuntu k8s.gcr.io
;; global options: +cmd
;; Got answer:
;; -HEADER- opcode: QUERY, status: NOERROR, id: 26261
;; flags: qr rd ad; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;k8s.gcr.io.                    IN      A

;; ANSWER SECTION:
k8s.gcr.io.             0       IN      CNAME   googlecode.l.googleusercontent.com.
googlecode.l.googleusercontent.com. 0 IN A      142.251.18.82

;; Query time: 10 msec
;; SERVER: 172.19.0.1#53(172.19.0.1)
;; WHEN: Wed Feb 23 11:23:00 UTC 2022
;; MSG SIZE  rcvd: 92

```

### traceroute for these repositories

k8s.gcr.io follows the standard path to the ISP and beyond

```zsh

root@kind-test-worker:/# traceroute k8s.gcr.io
traceroute to k8s.gcr.io (142.251.18.82), 30 hops max, 60 byte packets
 1  172.19.0.1 (172.19.0.1)  0.193 ms  0.014 ms  0.005 ms
 2  arunpc.mshome.net (172.25.112.1)  0.234 ms  0.214 ms  0.207 ms
 3  speedport.ip (192.168.1.1)  2.635 ms  2.583 ms  2.585 ms
```

However, `registry.docker.io`  and `registry-1.docker.io` fails

```zsh
root@kind-test-worker:/# traceroute registry.docker.io
registry.docker.io: Name or service not known
Cannot handle "host" cmdline arg `registry.docker.io` on position 1 (argc 1)

root@kind-test-worker:/# traceroute registry-1.docker.io
registry-1.docker.io: No address associated with hostname
Cannot handle "host" cmdline arg `registry-1.docker.io' on position 1 (argc 1)
```

traceroute to the ip address for registry-1.docker.com given in `ping` and also to a randomly selected IP from `dig` command works

```zsh
root@kind-test-worker:/# traceroute 34.200.175.181
traceroute to 34.200.175.181 (34.200.175.181), 30 hops max, 60 byte packets
 1  172.19.0.1 (172.19.0.1)  0.051 ms  0.006 ms  0.004 ms
 2  arunpc.mshome.net (172.25.112.1)  0.177 ms  0.172 ms  0.165 ms
 3  speedport.ip (192.168.1.1)  3.559 ms  3.617 ms  3.693 ms

root@kind-test-worker:/# traceroute 34.196.250.152
traceroute to 34.196.250.152 (34.196.250.152), 30 hops max, 60 byte packets
 1  172.19.0.1 (172.19.0.1)  0.120 ms  0.009 ms  0.005 ms
 2  arunpc.mshome.net (172.25.112.1)  0.206 ms  0.192 ms  0.185 ms
 3  speedport.ip (192.168.1.1)  2.752 ms  2.758 ms  2.915 ms

 ```

**So for some reason registry.docker.io and registry-1.docker.io seems to have issues with name resolution in the worker node, even though `dig` returns with NOERROR!!
**

</details>

## FINAL Solution

[After 2 weeks of trying to find solution](https://github.com/kubernetes-sigs/kind/issues/2645), I almost gave up. Then one day I came back and started again same commands. This time registry.docker.io was accessible and I could contiue to work!!!

NO IDEA what solved it.

[One option to try is to reset windows network config and restart](RemoteDev_docker_compose_image_Readme.md#network-issues-with-docker-containers). As kind is based on Docker, so this is relevant for Docker and not just Kind.

## Export the Kind image as WSL image

<details>

<summary>
Expand for details
</summary>

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

NOTE: Compressed image file is now about 700 MB.
</details>
