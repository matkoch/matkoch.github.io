# Setting up macOS for .NET Development

## Homebrew

```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

## Terminal

```
/bin/bash -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# https://github.com/romkatv/powerlevel10k
brew install romkatv/powerlevel10k/powerlevel10k
echo 'source /usr/local/opt/powerlevel10k/powerlevel10k.zsh-theme' >>! ~/.zshrc

brew cask install iterm2
brew install git svn

# https://github.com/Homebrew/homebrew-cask-fonts/tree/master/Casks
brew tap homebrew/cask-fonts
for font in \
    font-open-sans \
    font-roboto \
    font-meslolg-nerd-font
do
    brew cask install $font
done

p10k configure
```

## Browser

```
for browser in \
    firefox \
    google-chrome \
    brave-browser \
    microsoft-edge
do
    brew cask install $browser
done

default_browser="com.google.Chrome"
#default_browser = "com.mozilla.firefox"
#default_browser = "com.brave.Browser"
#default_browser = "com.microsoft.edgemac"

duti -s ${default_browser} http all
duti -s ${default_browser} https all
```

## Tools

```
declare -a brew_cask_install=(
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
for i in "${brew_cask_install[@]}"; do brew cask install $i; done
```



# Install Java

```
brew tap adoptopenjdk/openjdk
brew cask install adoptopenjdk8
```



# Setup VS Code

```
code --install-extension ms-dotnettools.csharp
code --install-extension cssho.vscode-svgviewer
```

# Settings?

```
defaults write com.apple.finder AppleShowAllFiles YES
defaults write com.apple.finder ShowPathbar -bool true
defaults write com.apple.finder ShowStatusBar -bool true
sudo spctl --master-disable
#sudo firmwarepasswd -setpasswd # https://github.com/T0mmykn1fe/DevSecOps-OSX-Mac-Setup-with-Homebrew
```

# Setup Default Apps (http://seriot.ch/resources/utis_graph/utis_graph.pdf)

```
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
```

# Clone Repositories

```
git clone git@github.com:matkoch/resharper-plugins ~/code/resharper-plugins
git clone git@github.com:matkoch/matkoch.github.io ~/code/blog
git clone git@github.com:matkoch/thumbnail-generator ~/code/thumbnail-generator
git clone git@github.com:nuke-build/nuke ~/code/nuke
dotnet tool install nuke.globaltool --global
echo 'export PATH=$HOME/.dotnet/tools:$PATH' >> ~/.zshrc
cd ~/code/nuke;              nuke generate-global-solution
cd ~/code/resharper-plugins; nuke generate-global-solution
```


# https://gist.github.com/squarism/ae3613daf5c01a98ba3a
# https://medium.com/@gveloper/using-iterm2s-built-in-integration-with-tmux-d5d0ef55ec30
# mas install 1254981365 # Contrast
# https://zellwk.com/blog/mac-setup-2/

# Disable SIP
echo "Reboot, hold ⌘+R, open terminal and type: csrutil disable"
```



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
    "dockutil"
