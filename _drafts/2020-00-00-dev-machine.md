Setting up macOS for .NET Development


#!/usr/bin/env bash

# Install ZSH
/bin/bash -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# Install Powerlevel10k // https://github.com/romkatv/powerlevel10k
brew install romkatv/powerlevel10k/powerlevel10k
echo 'source /usr/local/opt/powerlevel10k/powerlevel10k.zsh-theme' >>! ~/.zshrc

# Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"

declare -a brew_install=(
    "git"
    "svn"
    "mono"
    "tmux"              # https://github.com/tmux/tmux/wiki
    "imagemagick"       # https://imagemagick.org/index.php
    "tree"              # https://github.com/dbazile/gnu-tree-macos
    "htop"              # https://hisham.hm/htop/
    "ncdu"              # https://dev.yorhel.nl/ncdu
    "task"              # https://taskwarrior.org/
    "speedtest-cli"     # https://github.com/sivel/speedtest-cli
    "duti"              # https://github.com/moretension/duti/
    "mas"               # https://github.com/mas-cli/mas
)
declare -a brew_cask_install=(
    # Browser
    "google-chrome"
    "firefox"

    # Development Tools
    "dotnet-sdk"
    "jetbrains-toolbox"
    "iterm2"
    "docker"
    "visual-studio-code"

    # Connection
    "royal-tsx"
    "tunnelblick"
    "tripmode"

    # Social
    "ferdi"                     # https://getferdi.com/
    "slack"
    "skype"
    "telegram"

    # Productivity Tools
    "alfred"
    "bartender"
    "taskwarrior-pomodoro"
    "enpass"
    "bettertouchtool"
    "grammarly"
    "pock"
    "avibrazil-rdm"
    "dropbox"
    "turbo-boost-switcher"

    # Media Tools
    "spotify"
    "gimp"
    "vlc"
    "camtasia"
    "snagit"

    # Streaming
    "obs"
    "obs-virtualcam"
    "loopback"
    "streamlabs-obs"
)
for i in "${brew_install[@]}"; do brew install $i; done
for i in "${brew_cask_install[@]}"; do brew cask install $i; done

# Generate SSH Key // https://github.com/settings/keys
git config --global user.name "Matthias Koch"
git config --global user.email "ithrowexceptions@gmail.com"
ssh-keygen -t rsa -b 4096 -C "ithrowexceptions@gmail.com"
eval "$(ssh-agent -s)"
#touch ~/.ssh/config
ssh-add -K ~/.ssh/id_rsa
pbcopy < ~/.ssh/id_rsa.pub

# Install Java
brew tap adoptopenjdk/openjdk
brew cask install adoptopenjdk8

# Install Fonts // https://github.com/Homebrew/homebrew-cask-fonts/tree/master/Casks
declare -a brew_install_fonts=(
    "font-open-sans"
    "font-roboto"
    "font-source-code-pro"
    "font-source-sans-pro"
    "font-ubuntu"
)
brew tap homebrew/cask-fonts
for i in "${brew_install_fonts[@]}"; do brew cask install $i; done

# Setup VS Code
code --install-extension ms-dotnettools.csharp
code --install-extension cssho.vscode-svgviewer

# Settings
defaults write com.apple.finder AppleShowAllFiles YES
defaults write com.apple.finder ShowPathbar -bool true
defaults write com.apple.finder ShowStatusBar -bool true
sudo spctl --master-disable
#sudo firmwarepasswd -setpasswd # https://github.com/T0mmykn1fe/DevSecOps-OSX-Mac-Setup-with-Homebrew

# Default Apps (http://seriot.ch/resources/utis_graph/utis_graph.pdf)
duti -s com.microsoft.VSCode public.plain-text all
duti -s com.microsoft.VSCode public.unix-executable all
duti -s com.microsoft.VSCode public.data all

duti -s com.microsoft.VSCode .dotsettings all
duti -s com.microsoft.VSCode .targets all
duti -s com.microsoft.VSCode .props all
duti -s com.microsoft.VSCode .dotsettings all
duti -s com.microsoft.VSCode .cs all
duti -s com.microsoft.VSCode .csproj all

#duti -s com.apple.Safari public.html all
#$(osascript -e 'id of app "Visual Studio Code"')

# Clone Repositories
git clone git@github.com:matkoch/resharper-plugins ~/code/resharper-plugins
git clone git@github.com:matkoch/matkoch.github.io ~/code/blog
git clone git@github.com:matkoch/thumbnail-generator ~/code/thumbnail-generator
git clone git@github.com:nuke-build/nuke ~/code/nuke
dotnet tool install nuke.globaltool --global
echo 'export PATH=$HOME/.dotnet/tools:$PATH' >> ~/.zshrc
cd ~/code/nuke;              nuke generate-global-solution
cd ~/code/resharper-plugins; nuke generate-global-solution



# https://gist.github.com/squarism/ae3613daf5c01a98ba3a
# https://medium.com/@gveloper/using-iterm2s-built-in-integration-with-tmux-d5d0ef55ec30
# mas install 1254981365 # Contrast
# https://zellwk.com/blog/mac-setup-2/

# Disable SIP
echo "Reboot, hold ⌘+R, open terminal and type: csrutil disable"

