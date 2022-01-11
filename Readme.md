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

## Install WSL 2 debian

Do read [official documentation from Microsoft](https://docs.microsoft.com/en-us/windows/wsl/setup/environment#set-up-your-linux-username-and-password) for latest updates

Update wsl to ensure there is latest WSL kernel isntalled.

```bash
wsl --update
```

If you have not installed Debian already then install using command

```bash
wsl --install Debian --version 2
```

It will ask for user name to be setup. Provide the required information and you will be logged in the system with your account.

Intial insall size of the Debian VHDX is about 200-300MB.

Do check that you have latest version of Debian

```bash
cat /etc/os-release
```

At the time of writing "bullseye" or version 11 is the latest version of Debian OS.

## Upgrade Debian

If for whatever reason you have old version of Debian (I got by default Debian 9), then make sure to update it to the latest release. There are upgrade documentation made avilable by Debian for each version. make sure to follow the guides.

**NOTE**:

1. Make sure to follow the 2 step upgrade guide. It usually make this run smooth.
2. Debian has step by step upgrade from version to version. e.g. direct upgrade from 9 to 11 is not supported.

[Upgrade from Debian 9](https://www.debian.org/releases/buster/amd64/release-notes/ch-upgrading.en.html)

[Upgrade from Debian 10](https://www.debian.org/releases/stable/amd64/release-notes/ch-upgrading.en.html#system-status)

### Update to latest packages

Make sure you update to latest release of packages

```bash
sudo apt update && sudo apt upgrade -y
```

## Install some basic required packages

1. wget
2. curl
3. unzip

## Move WSL Debian to another HDD/SDD

I have about 150 GB on my C: drive. But for my experience, this is not an enough space especially if you want to have multiple WSL with additional softwares installed e.g. docker, kubernetes and other images.

At the time of this writing, this is no direct command to move a distribution to another location. This is acheived by export/import commands for WSL.

By default WSL distribution is installed on
%userprofile%\AppData\Local\Packages\TheDebianProject.DebianGNULinux....\LocalState\ext4.vhdx"

To move it to another location follow below steps. My choice below is D drive. Replace it with any other drive that you have.

1. Create a directory for your wsl distros virtual drives on the drive

   ```cmd
   cd d:

   mkdir wsl/debian11
   ```

2. Create a directory to store your wsl exports/backups
  
   ```cmd 
   mkdir wsl_backups
   ```

3. Shutdown WSL

   ```cmd
   wsl --shutdown
   
   wsl -l -v
   ```

4. Export old Debian. This will export the WSL2 distribution named Debian to the provided location.

   ```cmd
   wsl --export Debian ./wsl_backups/debian.tar
   ```

5. Import new Debian at new location. This will import the exported .tar file as a new Debian distribution named debian11 with .vhdx file stored at the given location.

   ```cmd
   import debian11 ./wsl/debian11 ./wsl_backups/debian.tar
   ```

6. Login to the new distribution. Information is same as your original system; in this case Debian

   ```cmd
   wsl -d debian11 -u "username"
   ```

7. Remove old distribution to clear up the space on C drive

   ```cmd
   wsl --unregister Debian
   ```

   NOTE: In case you want to keep the name of the distribution as original e.g. Debian then you need to execute step 7 before step 5.

8. Create new entry for the system (debian11) in Windows Terminal. This is quite easy. Simply exit Windows Terminal and start it again. You should have default entry created for it.
9. Set default user for the new distro. This is to avoid logging in as root user. Add/update [user] section in /etc/wsl.conf

    ```text
    [user]
    default = defaultusername
    ```

For detailed configuration parameters for WSL (global and per distribution) please check [Microsoft's official documentation](https://docs.microsoft.com/en-us/windows/wsl/wsl-config).

## Configure Windows Terminal to start Debian with default user and directory

By default WSL distros open with root user and in the directory from which you execute wsl or default user directory. There are different ways, but I prefered to configure Windows Terminal entry to configure the entry. Open Windows Terminal Settings and on the left side select your distribution. Set below options in **Tab General > commandline**

```text
wsl.exe ~ -d debian11 -u username
```

It opens the new session in home directory in Linux Distro given by "~" and with user "username". Do replace "username" with the user that you want to configure.

This setup gives the flexibility to configure mutiple Windwows Terminal sessions with different users and starting directories if one wants.

## Customize WSL2 terminal using Oh My Posh

This section is based on [these nicely documentated articles](https://www.ceos3c.com/wsl-2/windows-terminal-customization-wsl2-deep-dive/) by [Stefan](https://www.ceos3c.com/author/ceos3c_ic1l46/). Here I am just highlighting main points and issues I faced.

1. Install ZSH and set it as your default shell
2. Install [Oh My Posh](https://ohmyposh.dev/) created by [Jan De Dobbeleer](https://github.com/sponsors/JanDeDobbeleer)
3. Install [Nerd Fonts](https://www.nerdfonts.com/). This is done on Windows. I just installed Meslo LGM Nerd Font.
   - This was the tricky part for to get right.
   - It turns out that are some compaitbility issues between different OSes.
   - The [github repository for Nerd Fonts](https://github.com/ryanoasis/nerd-fonts/) provides all the various options and choices to install fonts.
   - After some trials I relized I had to install what is called "Windows compatible" Nerd Fonts.
   - Another challenge is to match the font name when it is installed with names given in the downloaded file names.
4. Set Meslo LGM Nerd Font in the Windows Terminal settings > Appearance > Font face. I set it up as default from my Windows Terminal. You can ofcourse select different font for different distro.
5. Activate Oh My Posh on ZSH. For this one need to add Oh My Posh inialization scripts with selected theme in `.zshrc` file

   ```bash
   # Oh My Posh Theme Config
   eval "$(oh-my-posh --init --shell zsh --config ~/.poshthemes/powerlevel10k_rainbow.omp.json)"
   ```

6. Customize Oh My Posh themes to your liking.
   There are variety of [themes available for Oh My Posh](https://ohmyposh.dev/docs/themes). You can also combine options from different themes by changing theme's json file. You can read about configuration possibilities on the [website](https://ohmyposh.dev/docs/config-overview).

## Git Setup

  Microsoft has [official documentation](https://docs.microsoft.com/en-us/windows/wsl/tutorials/wsl-git) on Git setup on WSL. Please read that for up-to-date information and further details.

### Install Git in WSL

  Git needs to be installed on each WSL distro, if we want to use WSL distro's filesystem as the project repositories.

### Install GCM (Git Credential Manager) inside WSL distro

  There are different ways to setup GCM in WSL environment.

- With Git for Windows
- Without Git for Windows

 Recommedation as per the [GCM documentation](https://github.com/GitCredentialManager/git-credential-manager/blob/main/docs/wsl.md) is to use with Git for Windows. However, in my case I had no plan to have development environment in Windows. For this reason only GCM is installed in Windows and 2nd option is chosen. Below steps are for the 2nd option.

 1. Install [GCM on Windows from the source](https://github.com/GitCredentialManager/git-credential-manager/releases/)
 2. Configure git config in WSL

   ```bash
   git config --global credential.helper "/mnt/c/Program\ Files\ \(x86\)/Git\ Credential\ Manager/git-credential-manager-core.exe"

   # For Azure DevOps support only
   git config --global credential.https://dev.azure.com.useHttpPath true
   ```

3. In windows set envoronment variable WSLENV. From and Administrator cmd prompt execute

 ```cmd
    SETX WSLENV %WSLENV%:GIT_EXEC_PATH/wp
 ```

 4. Restart WSL and Windows

## Oh My Posh in action

I skipped earlier on what happens when Oh My Posh is installed. You might have read the documentation earlier to know what it is about. Here I am capturing some results for the sake of completeness on my setup.

1. Default shell with OMP: A nice look with execution duration
   ![simple Oh My Posh in action](images/omp.drawio.svg)

2. With Git extension installed and repository fully synched
   ![Git repo fuly synched](images/omp_git_synch.drawio.svg)
3. With Repository not fully synched. 2 additions, 1 update and 1 deletion.
   ![Git repo with unsync changes](images/omp_git_notsynch.drawio.svg)

## Remote WSL development using Visual Studio code

1. [Install Visual Studio Code](https://code.visualstudio.com/download) in Windows.
2. [Install Remote Development Extension pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack). If you are not going to use docker or other remote SSH server then you can only install Remote WSL extension.
3. Open your project folder in WSL distro in VS Code. On your WSL shell in the appropriate folder execute

   ```bash
   code .
   ```

4. Alternatively one can also use Remote WSL extension commands from VS code command palette open a certain folder for development.
5. When VS Code is started for the first time in WSL distro it will installe VS Code server in WSL distro to allow connections from VS Code client from Windows.
6. Install extensions inside VS code remote. This is needed based on the extension. e.g. if you use python then Python extension needs to be installed on each WSL distibution. Some extensions are only required to be installed on Windows. VS Code prompts and give option to WSL install for applicable extensions.

   **NOTE**: When you open terminal window in Remote WSL session. VS Code opens terminal with default user configured for that WSL distro, which is usually root user. You can either switch to your desired user in terminal for commands or set the default user to the required one before opening VS Code Remote WSL session. [Currently there is no option to select/configure a user per session/workspace./project.](https://github.com/microsoft/vscode-remote-release/issues/286)

## Remote Docker development on WSL using Visual Studio Code

As mentioned in the motivation I wanted to have linux as the main development environment, so I decided not to install Docker Desktop on Windows, rather inside WSL. Microsoft has [official documentation](https://code.visualstudio.com/docs/remote/containers) with all the requirements and options available.

To configure Docker and Docker compose, I followed the [this article](https://dev.to/felipecrs/simply-run-docker-on-wsl2-3o8) from [Felipe Santos](https://dev.to/felipecrs).

NOTE: Below steps are based on image created above. For my case, I followed below steps to keep these as 2 seprate images

```cmd
wsl --export debian11 ./wsl_backups/deb11base.tar
mkdir wsl/deb11docker
wsl --import deb11docker ./wsl/deb11docker ./wsl_backups/deb11base.tar
wsl -d deb11docker
```

Note: Exported image size is now about 1GB.

Now I am logged in the new WSL distro for further configuration

### Setup Docker inside WSL2

1. Install Docker CE for Debian [following official documentation](https://docs.docker.com/engine/install/debian/)

   NOTE: For me the official instructions were missing the step to start the daemon before I could execute the test command i.e. `docker run hello-world`. I add that step below to avoid surprises.

   ```zsh
   # Remove old packages
   sudo apt-get remove docker docker-engine docker.io containerd runc

   # Install pre-requisites
   sudo apt-get update

   sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

   # Add docker official gpg key
   curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

   # Add dockers stable repository for relevant architecture and release version
   echo \
   "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
   $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

   # Update apt repos
   sudo apt update 

   # Installed Docker engine
   sudo apt-get install docker-ce docker-ce-cli containerd.io
   ```

2. Perform [post installation steps](https://docs.docker.com/engine/install/linux-postinstall/)

   ```zsh

   # Add Docker group
   sudo groupadd docker

   # Add required user to docker group
   sudo usermod -aG docker $USER

   ```

3. Add /etc/fstab file. For some reason Docker needs it and it was not present on Debian

   ```zsh
   sudo touch /etc/fstab
   ```

4. Update alternatives for iptables as below. I am not sure what exactly it does. But I found this solution in the issue on github.

   ```zsh
   sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
   sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
   ```

5. Make Docker Deamon start on WSL initialization. Add below lines in the /etc/wsl.conf file

   ```zsh
   [boot]
   command = service docker start
   ```

6. Exit and restart WSL distro.
7. Test if docker can run without root. It should run now unless some other issues has occured.

   ```zsh
   docker version
   
   docker run hello-world
   ```

### Setup Docker Compose (V2) inside WSL2

This part was bit tricky for to understand. It turns out that with  [Docker compose V2](https://github.com/docker/compose/tree/v2#linux) implementation was drastically changed and not just the implementation language from Python to golang. 

Due to this to keep all 3rd party application integrations, what still used old commands, working an intermediary application was developed called [compose-switch](https://github.com/docker/compose-switch). It basically translate old commands to new commands. Overall installation steps are as follow.

1. Install Docker Compose v2 at system level. There is an option to install at user level as well. One just need to change the location of the file `~/.docker/cli-plugins/docker-compose` for it.

   ```zsh
   # Finds the latest version
   $ compose_version=$(curl -fsSL -o /dev/null -w "%{url_effective}" https://github.com/docker/compose/releases/latest | xargs basename)

   # Downloads the binary to the plugins folder
   $ sudo curl -fL --create-dirs -o /usr/libexec/docker/cli-plugins/docker-compose \
      "https://github.com/docker/compose/releases/download/${compose_version}/docker-compose-linux-$(uname -m)"

   # Assigns execution permission to it
   $ chmod +x /usr/libexec/docker/cli-plugins/docker-compose
   ```

2. Test if Docker compose V2 is working

   ```zsh
   docker compose version
   ```

3. Install compose-switch. This based on the article linked in the beginning for VS Code. I assume VS Code still use docker-compose commands somewhere. This was bit tricky as I found small differences in [official documentation](https://github.com/docker/compose-switch) and the [article I followed](https://dev.to/felipecrs/simply-run-docker-on-wsl2-3o8). Official documentation has an install script, but it could not setup the alternatives due to permissions. I followed below manual steps to get it installed.

   1. Use installation script from the installation script to install latest compose-switch.

   2. Set alternative for docker-compose to point it to compose-switch. This internally with then translate old version commands to new version.

      ```zsh
      sudo update-alternatives --install /usr/local/bin/docker-compose docker-compose /usr/local/bin/compose-switch 99
      ```

   3. Test docker-compose and docker compose are same

      ```zsh
      docker compose version
      docker-compose version
      ```

   4. Install Docker credential helper. This is needed to store docker credential if using `docker login`. 

      ```zsh
         # Finds the latest version
         $ wincred_version=$(curl -fsSL -o /dev/null -w "%{url_effective}" https://github.com/docker/docker-credential-helpers/releases/latest | xargs basename)

         # Downloads and extracts the .exe
         $ sudo curl -fL \
            "https://github.com/docker/docker-credential-helpers/releases/download/${wincred_version}/docker-credential-wincred-${wincred_version}-$(dpkg --print-architecture).zip" |
            zcat | sudo tee /usr/local/bin/docker-credential-wincred.exe >/dev/null

         # Assigns execution permission to it
         $ sudo chmod +x /usr/local/bin/docker-credential-wincred.exe
      ```

   5. Configure docker so that docker cli can use it. Edit ~/.docker/config.json to add below entries. This will help share Docker crendtials among different WSL distros using Windows Credential manager

      ```json
      {
         "credsStore": "wincred.exe"
      }
      ```

   6. Test Crdential helper setting by logging into to docker. If there is no complain about plain text password then it has worked. Or if it already used credentials from Windows.

      ```zsh
      docker login
      ```

   7. Enable [BuildKit feature](https://docs.docker.com/develop/develop-images/build_enhancements/). Add below lines in `/etc/docker/daemon.json`

      ```json
      {
         "features": {
            "buildKit": true
         }
      }
      ```

### Export the image

 As next base image for Remote Container development. Shutdown the WSL distro first
  
  ```powershell
   
   wsl --export deb11docker ./wsl_backups/deb11_docker.tar
   ```

NOTE: Image size is now about 1.5GB

### Test Docker development container using VS Code

WSL Debian environment is setup with 

- Git, Docker CE and Docker Compose V2 in WSL
  
- Git Credential Manager (GCM) and Visual Studio Code with Remote Extension pack in Windows.