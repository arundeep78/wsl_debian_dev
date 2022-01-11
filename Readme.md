# WSL2 debian 11 (Bullseye) setup on Windows 11 for development

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
 `wsl -install Debian`

 Make sure you have WSL version 2 set as the default version.

 `wsl --set-default-version 2`

By default Ubuntu will be installed.

## Install WSL 2 debian

Do read [official documentation from Microsoft](https://docs.microsoft.com/en-us/windows/wsl/setup/environment#set-up-your-linux-username-and-password) for latest updates


Update wsl to ensure there is latest WSL kernel isntalled.

`wsl --update`

If you have not installed Debian already then install using command
`wsl --install Debian --version 2`

It will ask for user name to be setup. Provide the required information and you will be logged in the system with your account.

Intial insall size of the Debian VHDX is about 200-300MB.

Do check that you have latest version of Debian
`cat /etc/os-release`
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

`sudo apt update && sudo apt upgrade -y`

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
   
   `cd d:`

   `mkdir wsl/debian11`
2. Create a directory to store your wsl exports/backups
  
   `mkdir wsl_backups` 
3. Shutdown WSL
   
   `wsl --shutdown`

   `wsl -l -v`
4. Export old Debian. This will export the WSL2 distribution named Debian to the provided location. 
   
   `wsl --export Debian ./wsl_backups/debian.tar` 

5. Import new Debian at new location. This will import the exported .tar file as a new Debian distribution named debian11 with .vhdx file stored at the given location.
   
   `import debian11 ./wsl/debian11 ./wsl_backups/debian.tar`

6. Login to the new distribution. Information is same as your original system; in this case Debian
   
   `wsl -d debian11 -u "username"`
7. Remove old distribution to clear up the space on C drive
   
   ` wsl --unregister Debian`

NOTE: In case you want to keep the name of the distribution as original e.g. Debian then you need to execute step 7 before step 5.

8. Create new entry for the system (debian11) in Windows Terminal. This is quite easy. Simply exit Windows Terminal and start it again. You should have default entry created for it.
9. Set default user for the new distro. This is to avoid logging in as root user. Add/update [user] section in /etc/wsl.conf
    ```
    [user]
    default = defaultusername
    ```

For detailed configuration parameters for WSL (global and per distribution) please check [Microsoft's official documentation](https://docs.microsoft.com/en-us/windows/wsl/wsl-config).

## Configure Windows Terminal to start Debian with default user and directory
By default WSL distros open with root user and in the directory from which you execute wsl or default user directory. There are different ways, but I prefered to configure Windows Terminal entry to configure the entry. Open Windows Terminal Settings and on the left side select your distribution. Set below options in **Tab General > commandline**

`wsl.exe ~ -d debian11 -u username`


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
   
   ```
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
   ```
   git config --global credential.helper "/mnt/c/Program\ Files\ \(x86\)/Git\ Credential\ Manager/git-credential-manager-core.exe"

      # For Azure DevOps support only
      git config --global credential.https://dev.azure.com.useHttpPath true
   ```
 3. In windows set envoronment variable WSLENV. From and Administrator cmd prompt execute
 ```
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

##   Remote WSL development using Visual Studio code
1. [Install Visual Studio Code](https://code.visualstudio.com/download) in Windows. 
2. [Install Remote Development Extension pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack). If you are not going to use docker or other remote SSH server then you can only install Remote WSL extension.
3. Open your project folder in WSL distro in VS Code. On your WSL shell in the appropriate folder execute 
   ```
   code .
   ```
4. Alternatively one can also use Remote WSL extension commands from VS code command palette open a certain folder for development.
5. When VS Code is started for the first time in WSL distro it will installe VS Code server in WSL distro to allow connections from VS Code client from Windows.
6. Install extensions inside VS code remote. This is needed based on the extension. e.g. if you use python then Python extension needs to be installed on each WSL distibution. Some extensions are only required to be installed on Windows. VS Code prompts and give option to WSL install for applicable extensions.
   
   **NOTE**: When you open terminal window in Remote WSL session. VS Code opens terminal with default user configured for that WSL distro, which is usually root user. You can either switch to your desired user in terminal for commands or set the default user to the required one before opening VS Code Remote WSL session. [Currently there is no option to select/configure a user per session/workspace./project.](https://github.com/microsoft/vscode-remote-release/issues/286) 
 