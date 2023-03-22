# Remote Docker development on WSL using Visual Studio Code

Make sure to read the [main Readme documentation](Readme.md) and that you have all pre-requisites installed.

NOTE: Below steps are done using [Base development image](base_image_Readme.md). If you are using just vanilla Debian OS, then make sure to follow the steps in the Base image development, otherwise below steps won't achieve the desired result.

As mentioned in the motivation I wanted to have linux as the main development environment, so I decided not to install Docker Desktop on Windows, rather inside WSL. Microsoft has [official documentation](https://code.visualstudio.com/docs/remote/containers) with all the requirements and options available.

To configure Docker and Docker compose, I followed the [this article](https://dev.to/felipecrs/simply-run-docker-on-wsl2-3o8) from [Felipe Santos](https://dev.to/felipecrs).

Assuming you have an existing base image available as tar file. Create a new WSL environment by importing that image as starting point.

```cmd
mkdir wsl/deb11docker
wsl --import deb11docker ./wsl/deb11docker ./wsl_backups/deb11base.tar
wsl -d deb11docker
```

Now I am logged in the new WSL distro for further configuration

## Setup Docker inside WSL2

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

## Setup Docker Compose (V2) inside WSL2

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
   $ sudo chmod +x /usr/libexec/docker/cli-plugins/docker-compose
   ```

2. Test if Docker compose V2 is working

   ```zsh
   docker compose version
   ```

3. Install compose-switch. This based on the article linked in the beginning for VS Code. I assume VS Code still use docker-compose commands somewhere. This was bit tricky as I found small differences in [official documentation](https://github.com/docker/compose-switch) and the [article I followed](https://dev.to/felipecrs/simply-run-docker-on-wsl2-3o8). Official documentation has an install script, but it could not setup the alternatives due to permissions. I followed below manual steps to get it installed.

   1. Use installation script from the installation script to install latest compose-switch.

      ```zsh
      curl -fLO https://raw.githubusercontent.com/docker/compose-switch/master/install_on_linux.sh

      chmod +x install_on_linux.sh
      sudo ./install_on_linux.sh
      ```

   It copied compose-switch to the required path but failed to set alternatives.

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

## Add docker plugin for Oh My ZSH

Add below plugins to `~./zshrc` to enable autocompletions, if you are using Oh My ZSH

   ```zsh
   plugins=(... docker)
   ```

## Install VS Code extension

Install [Docker extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-docker) in VS Code. Make sure it is installed inside WSL.

## Export the image

 As next base image for Remote Container development. Shutdown the WSL distro first
  
  ```powershell
   
   wsl --export deb11docker ./wsl_backups/deb11_docker.tar | tar -czf ./wsl_backups/deb11_docker.tar.gz ./wsl_backups/deb11_docker.tar

   rm ./wsl_backups/deb11_docker.tar
   ```

NOTE: Image size is now about 500 MB

## Test Docker development container using VS Code

WSL Debian environment is setup with

- Git, Docker CE and Docker Compose V2 in WSL
  
- Git Credential Manager (GCM) and Visual Studio Code with Remote Extension pack in Windows.

Now let's test if this image is working for development container using VS Code. [Official documentaion](https://code.visualstudio.com/docs/remote/containers) provides further details on options and maintenance of the containers. Below steps are for python development environment.

1. Login to WSL and create a folder for test project and open it in VS Code

   ```zsh
   mkdir projects/test_devcontainer
   cd projects/test_devcontainer
   code .
   ```

2. Create a new test python file in VS Code `hello_world.py` with simple print statement.

   ```python
   print("hello world!")
   ```

3. Create a Use VS Code Command  `Remote-Containers: Reopen Folder in Container`

   ![VS Code command platette Remote containers : reopen in container](images/devcontainer/open_in_remote_container.drawio.svg)

4. Select the base image. You will be prompted for it, if you have not already initialzed this project with remote container or there is no `.devcontainer` folder and `.devcontainer.json` files in it. For this test project I selected `Python3`
   ![Select base image for devcontainer](images/devcontainer/select_base_image.drawio.svg)
5. Select version of `Python`. I went with default i.e. 3.10
   ![Select python version](images/devcontainer/select_python_version.drawio.svg)
6. Select if you want to install node.js. Selected `None`
   ![Select if to install Node.js or not and its version](images/devcontainer/ignore_node.js.drawio.svg)
7. Any additional features to install on the image. Selected `None` and click Ok.
   ![Select additional features to install](images/devcontainer/additional_features.drawio.svg)
8. VS Code will trigger the build of the new docker image based on selected parameters and will start the container when ready. You can see the logs by clicking the prompt in the bottom-right corner.
   ![Check logs of the build container logs](images/devcontainer/build_container_logs.drawio.svg)
9. Once the container is ready, you can see change in the prompt in the bottom-left corner.
    ![Remote container is displayed in bottom-left corner](images/devcontainer/remote_container_name_vscode.drawio.svg)
10. Folder `.devcontainer` is created with a DockerFile and devcontainer.json file. Name of the devcontainer as defined in devcontainer.json file is also visible in the Explorer.
    ![devcontainer folder structure in VS Code](images/devcontainer/devcontainer_structure.drawio.svg)
11. Run the `hello_world.py` using VScode command palette `Run Python File in Terminal`
12. You will see the output in Terminal session. It is working!!!
    ![Output should be visible in the terminal](images/devcontainer/output_python.drawio.svg)

## Network issues with Docker containers

It happens time to time that inside running containers, applications cannot connect to network and will fail. e.g.

```bash
sudo apt-get update

Err:1 http://archive.ubuntu.com/ubuntu focal InRelease
  Temporary failure resolving 'archive.ubuntu.com'
Err:2 http://security.ubuntu.com/ubuntu focal-security InRelease
  Temporary failure resolving 'security.ubuntu.com'
Err:3 http://archive.ubuntu.com/ubuntu focal-updates InRelease
  Temporary failure resolving 'archive.ubuntu.com'

```

One solution is try to reset windows network and restart windows

```cmd
wsl --shutdown
netsh winsock reset
netsh int ip reset all
netsh winhttp reset proxy
ipconfig /flushdns
```
