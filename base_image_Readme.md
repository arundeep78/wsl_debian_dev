
# Base WSL development environment based on Debian 11

Make sure to read the main [Readme documentation](Readme.md) and ensure you have pre-requisites installed.

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

Intial install size of the Debian VHDX is about 200-300MB.

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
4. lsb-release

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

## Install ZSH

1. Install ZSH and set it as your default shell

   ```zsh
   sudo apt install zsh -y
   chsh -s $(which zsh)
   ```

2. Logout, close WSL session and login in again

## Git Setup

  Microsoft has [official documentation](https://docs.microsoft.com/en-us/windows/wsl/tutorials/wsl-git) on Git setup on WSL. Please read that for up-to-date information and further details.

### Install Git in WSL

  Git needs to be installed on each WSL distro, if we want to use WSL distro's filesystem as the project repositories.

  ```zsh
  sudo apt install git -y
  ```

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

## Install Github CLI

To know more, read [official documentation](https://cli.github.com/) from GitHub. [Linux specific installation instructions](https://github.com/cli/cli/blob/trunk/docs/install_linux.md) are available at github.

```zsh
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null

sudo apt update

sudo apt install gh
```

## Install Github Large file support

Provide support for [large file support for github repos](https://github.com/git-lfs/git-lfs/wiki/Installation).

```bash
   curl -s https://packagecloud.io/install/repositories/github/git-lfs/script.deb.sh | sudo bash

   sudo apt-get install git-lfs
   git lfs install
````

Now one can configure large files to be handles by github LFS. You can manually add files int the repo with `git lfs track *.psd`  or you can edit `.gitattributes` file

# Install and configure Oh My ZSH

Rather than me saying anything about it, just read on [Oh My ZSH website](https://ohmyz.sh/). Atleast for me, with its configurable plugins and themes, it made working on Linux shell easy and intersting. I am not one of those geeks who like to remember every command and like black an white screen!

1. Install pre-requisities. if you have followed all the steps above then pre-requistes are already installed.
2. Install Oh My ZSH

   ```zsh
   sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
   ```

3. [Install Powerlevel10K theme](https://github.com/romkatv/powerlevel10k)
   1. [install recommended font](https://github.com/romkatv/powerlevel10k#meslo-nerd-font-patched-for-powerlevel10k) - Meslo Nerd font
   2. Install powerlevel10k theme

   ```zsh
      # Clone the repository
      git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
      ```

   3. Set `ZSH_THEME="powerlevel10k/powerlevel10k"` in `~/.zshrc`.
   4. Logout and login into the sheel to activate it. On can also execute `p10k configure` to start the configuration wizard. Follow your preferences and configure the prompt. I went for `rainbow` style.
   ![Oh My ZSH powerline10k rainbow prompt](images/baseimage/omz_prompt.drawio.svg)

4. Configure [plugins for Oh My ZSH](https://github.com/ohmyzsh/ohmyzsh/wiki/Plugins). Add below plugins to `~./zshrc` to enable autocompletions

   ```zsh
   plugins=(git gh)
   ```

5. Install [zsh auto-suggestions](https://github.com/zsh-users/zsh-autosuggestions). Based on the installation type procedure may change. For Oh My ZSH method is below.  

   1. Copy the plugin

      ```zsh
      git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions 
      ```

   2. Add plugin in the `.zshrc` file

      ```zsh
      plugins=( 
         # other plugins...
         zsh-autosuggestions
         )
      ```

   3. Based on your terminal color layout, you might have configure parameter `ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE`. For Oh My ZSH, you can edit that parameter in `$ZSH_CUSTOM/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh`. 

## Customize WSL2 terminal using Oh My Posh

This is another way to beautify your terminal. Oh My Posh seems to cover only themes, while oh My ZSH covers themes and plugins. Both can work together as well. You can make your choice based on your preferences.  

This section is based on [these nicely documentated articles](https://www.ceos3c.com/wsl-2/windows-terminal-customization-wsl2-deep-dive/) by [Stefan](https://www.ceos3c.com/author/ceos3c_ic1l46/). Here I am just highlighting main points and issues I faced.

1. [Install ZSH](#install-zsh) and set it as your default shell, if not already done.

2. Install [Oh My Posh](https://ohmyposh.dev/) created by [Jan De Dobbeleer](https://github.com/sponsors/JanDeDobbeleer)

   ```zsh
   sudo wget https://github.com/JanDeDobbeleer/oh-my-posh/releases/latest/download/posh-linux-amd64 -O /usr/local/bin/oh-my-posh

   sudo chmod +x /usr/local/bin/oh-my-posh
   ```

3. Download themes

   ```zsh
   mkdir ~/.poshthemes

   wget https://github.com/JanDeDobbeleer/oh-my-posh/releases/latest/download/themes.zip -O ~/.poshthemes/themes.zip

   unzip ~/.poshthemes/themes.zip -d ~/.poshthemes

   chmod u+rw ~/.poshthemes/*.json

   rm ~/.poshthemes/themes.zip
   ```

4. Install [Nerd Fonts](https://www.nerdfonts.com/). This is done on Windows. I just installed Meslo LGM Nerd Font.
   - This was the tricky part to get right.
   - It turns out that are some compaitbility issues between different OSes.
   - The [github repository for Nerd Fonts](https://github.com/ryanoasis/nerd-fonts/) provides all the various options and choices to install fonts.
   - After some trials I relized I had to install what is called "Windows compatible" Nerd Fonts.
   - Another challenge is to match the font name when it is installed with names given in the downloaded file names.
5. Set Meslo LGM Nerd Font in the Windows Terminal settings > Appearance > Font face. I set it up as default from my Windows Terminal. You can ofcourse select different font for different distro.
6. Activate Oh My Posh on ZSH. For this one need to add Oh My Posh inialization scripts with selected theme in `.zshrc` file

   ```bash
   # Oh My Posh Theme Config
   eval "$(oh-my-posh --init --shell zsh --config ~/.poshthemes/powerlevel10k_rainbow.omp.json)"
   ```

   If you have Oh-my-ZSH also installed then comment out the below line in `~/.zshrc`

   ```zsh
   # ZSH_THEME="powerlevel10k/powerlevel10k"
   ```

7. Customize Oh My Posh themes to your liking.
   There are variety of [themes available for Oh My Posh](https://ohmyposh.dev/docs/themes). You can also combine options from different themes by changing theme's json file. You can read about configuration possibilities on the [website](https://ohmyposh.dev/docs/config-overview).

## Oh My Posh in action

NOTE: visualizations are valid for both Oh My ZSH and Oh my Posh, based on your configuration.

You might have read the documentation earlier to know what it is about. Here I am capturing some results for the sake of completeness on my setup.

1. Default shell with OMP: A nice look with execution duration
   ![simple Oh My Posh in action](images/baseimage/omp.drawio.svg)

2. With Git extension installed and repository fully synched
   ![Git repo fuly synched](images/baseimage/omp_git_synch.drawio.svg)
3. With Repository not fully synched. 2 additions, 1 update and 1 deletion.
   ![Git repo with unsync changes](images/baseimage/omp_git_notsynch.drawio.svg)

## Remote WSL development using Visual Studio code

1. [Install Visual Studio Code](https://code.visualstudio.com/download) in Windows.
2. [Install Remote Development Extension pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack). If you are not going to use docker or other remote SSH server then you can only install Remote WSL extension.
3. Open your project folder in WSL distro in VS Code. On your WSL shell in the appropriate folder execute

   ```zsh
   code .
   ```

4. Alternatively one can also use Remote WSL extension commands from VS code command palette open a certain folder for development.
5. When VS Code is started for the first time in WSL distro it will installe VS Code server in WSL distro to allow connections from VS Code client from Windows.
6. Install extensions inside VS code remote. This is needed based on the extension. e.g. if you use python then Python extension needs to be installed on each WSL distibution. Some extensions are only required to be installed on Windows. VS Code prompts and give option to WSL install for applicable extensions.

   **NOTE**: When you open terminal window in Remote WSL session. VS Code opens terminal with default user configured for that WSL distro, which is usually root user. You can either switch to your desired user in terminal for commands or set the default user to the required one before opening VS Code Remote WSL session. [Currently there is no option to select/configure a user per session/workspace./project.](https://github.com/microsoft/vscode-remote-release/issues/286)

## Export WSL image for later use

Export the WSL Debian image. Other images can then be build on top of this image.

   ```cmd
   wsl --export debian11 ./wsl_backups/deb11base.tar | tar -czf ./wsl_backups/deb11base.tar.gz ./wsl_backups/deb11base.tar

   ```

Note: Exported image size is now about 300MB.
