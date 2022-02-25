# WSL2 image with Microk8s installation

This documentation to have configure a WSL2 Debian 11 image with microk8s installed on the image. This is based on the [standard Debian 11 image configured with basic tools like Oh-my-ZSH etc](base_image_Readme.md).

I decided to create a separate image simply not to have any conflict with other flavours of K8s. Microk8s uses snap package and I read somewhere that one of the flavours(I think it was Kind) does not like snap. Maybe those issues are solved, but I already had issues with Kind installtion, so lets continue to have a isolation. Afterall all this containerization is about isolation, right?


## Why MicroK8s?

The documentation says that it has a very low memory footprint. This is cool as it should help keep some resources free on my laptop. 

It was also the image mentioned in one of the [Microsoft Tutorails](https://docs.microsoft.com/en-us/learn/modules/intro-to-kubernetes/) that I followed to learn K8s. 

Additionally, it does not seems to need docker to run it. At the moment, I have [issues with Kind and Minikube clusters](https://github.com/arundeep78/wsl_debian_dev/blob/master/Kind_k8s_Readme.md#check-internet-access-from-nodes) as clusters nodes cannot reach docker repository, even though all other internet sites are accessible! So, it might help me to get going with actual cluster tasks rather than figuring out why I cannot reach docker repository.

## Create a new WSL environment

 Execute below commands to create starting WSL image to develop Microk8s WSL2 image

```powershell
wsl --import deb11mk8s .\wsl\deb11_mk8s\ .\wsl_backup\deb11_base.tar
```

One must change name of images and paths based on the specific setup.

## Install SNAP - as per SNAP document

Debian uses apt as package manager by default. For some reason MicroK8s is distributed only through SNAP and not by apt. [Install SNAP for your distribution](https://snapcraft.io/docs/installing-snapd) using relevant instructions.

1. Install SNAP

    ```zsh
    sudo apt update
    sudo apt install snapd
    ```  

2. Logout and login to the session again
3. Install snap core 
   
    ```zsh
    $sudo snap install core
    error: cannot communicate with server: Post "http://localhost/v2/snaps/core": dial unix /run/snapd.socket: connect: no such file or directory
    ```
  
  ### **The issue** : Apparently SNAP does not work so easily with WSL2.

  There are many issues raised regarding this. One jsut need to make a simple google search. Different workarounds are proposed with [scripts](https://github.com/Microsoft/WSL/issues/2374), [custom kernels](https://forum.snapcraft.io/t/running-snaps-on-wsl2-insiders-only-for-now/13033) and various package and [start scripts](https://github.com/microsoft/WSL/issues/5126#issuecomment-65371520) 


  This is way too geeky for my liking. especially now that it also involves being specific to shell and its related script e.g. bash vs zsh! Thankfully I found other articles, that explained similar scripts with proper details

  
  ### **Uninstall SNAP** to start clean again
  
  ```zsh
  sudo apt remove snapd
  ```

## Install Microk8s on WSL2 on Windows 11 using special procedure

  3 years since a workaround was developed and yet, no standard solution for WSL2. Below is the article chain that I happend to come across and tried.

  - [Deploy MicroK8s on WSL2 by JakeVis](https://jv.ag/blog/Deploy-MicroK8s-On-WSL2/). Published 2021
  
  - It is further based on 
  [WSL2 _Microk8s by Nunix](https://wsl.dev/wsl2-microk8s/) which is further based on - published in 2020

  - [Running SNAPs on WSL2 - Insiders only for now](https://forum.snapcraft.io/t/running-snaps-on-wsl2-insiders-only-for-now/13033) - published in 2019. 
  - This in turn is now abandoned and has been moved to a script maintained at [ubuntu wsl2 systemd script](https://github.com/damionGans/ubuntu-wsl2-systemd-script)
  - I also found an [ansible automated script](https://gist.github.com/ct27stf/77582676a42e2f409ccd227773393623#file-setupmicrok8s-sh). I did not use because I wanted to have atleast some understanding of what is happening in all those changes.

### Using manual steps/scripts as listed in 1st 3 links: failed

1. Install required packages for SystemD
   
    ```zsh
    sudo apt install -yqq fontconfig daemonize
    ```
2. Check ids of user and groups
   
   ```zsh
   $id
   uid=1000(arundeep) gid=1000(arundeep) groups=1000(arundeep),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev)
   ```

3. Update `/etc/wsl.conf` with below content. make sure to use Ids from the command above and username is your username. I commented out some entries below as there are default. Also commented out crossDistro as per remark in [github 4577](https://github.com/microsoft/WSL/issues/4577#issuecomment-685173446) in Sept 2020!. I don't know if other options are also useless/changed now!!
  
   ```zsh
   [automount]
    #enabled = true  # default true
    options = "metadata,umask=22,fmask=11" #"uid=1000,gid=1000,case=off" these are default
    #mountFsTab = true  # is default true
    #crossDistro = true

    [network]
    generateHosts = false
    #generateResolvConf = true # default true

    [interop]
    #enabled = true # default true
    #appendWindowsPath = true # default true
   ```

4. Create starting script for SystemD `/etc/profile.d/00-wsl2-systemd.sh`. For some reason this script looks much simpler than what the automated script contains. let's see how it goes. Maybe, I would have to use automated script afterall
   
   ```zsh
    SYSTEMD_PID=$(ps -ef | grep '/lib/systemd/systemd --system-unit=basic.target$' | grep -v unshare | awk '{print $2}')

    if [ -z "$SYSTEMD_PID" ]; then
      sudo /usr/bin/daemonize /usr/bin/unshare --fork --pid --mount-proc /lib/systemd/systemd --system-unit=basic.target
      SYSTEMD_PID=$(ps -ef | grep '/lib/systemd/systemd --system-unit=basic.target$' | grep -v unshare | awk '{print $2}')
    fi

    if [ -n "$SYSTEMD_PID" ] && [ "$SYSTEMD_PID" != "1" ]; then
        exec sudo /usr/bin/nsenter -t $SYSTEMD_PID -a su - $LOGNAME
    fi
   ```

5. Setup netowork forwarding. Automated script does not set this up
   
   ```zsh
   echo 'net.ipv4.conf.all.route_localnet = 1' | sudo tee -a /etc/sysctl.conf
   sudo sysctl -p /etc/sysctl.conf
   ```

6. Restart WSL

7. Install `snap`
   
   ```zsh
   sudo apt install snapd -y
   ```

8. check snap. Fails!!
   
   ```zsh
    $snap version
    snap    2.49-1+deb11u1
    snapd   unavailable
    series  -
   
   $snap list
    error: cannot list snaps: cannot communicate with server: Get "http://localhost/v2/snaps": dial unix /run/snapd.socket: connect: no such file or directory   
   ```

9. Clean up above steps
   1. Remove snap
   2. remove ip4 forwarding from `/etc/sysctl.conf`
   3. remove `/etc/profile.d/00-wsl2-systemd.sh` file
   4. revert back changes to `/etc/wsl.conf`
   5. remove `fontconfig` and `daemonize`
10. Execute below command as per the ansible script github
   
   ```zsh
   curl -sf -L https://gist.github.com/ct27stf/77582676a42e2f409ccd227773393623/raw/SetupMicroK8s.sh | sh
   ```
   
   
   That's where I stop! The other script is also setting up some variables in Windows, which I am not sure if apply to all Distros or is there a way to isolate it to a particular WSL distro. As I have other WSL distros running in parallel, I stopped this pursuit. I might try to do that in a VirtualBox VM later!!

   

