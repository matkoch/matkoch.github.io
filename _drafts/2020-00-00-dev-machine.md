# Setup macOS environment

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

# https://github.com/Homebrew/homebrew-cask-fonts/tree/master/Casks
brew tap homebrew/cask-fonts
brew cask install font-roboto
brew cask install font-cascadia
brew cask install font-fira-code
brew cask install font-source-code-pro
brew cask install font-firacode-nerd-font
brew cask install font-sourcecodepro-nerd-font
brew cask install font-meslolg-nerd-font

p10k configure
brew cask install iterm2

brew install git
brew install svn
brew install tmux                 # https://github.com/tmux/tmux/wiki
brew install tree                 # https://github.com/dbazile/gnu-tree-macos
brew install htop                 # https://hisham.hm/htop/
brew install ncdu                 # https://dev.yorhel.nl/ncdu
brew install task                 # https://taskwarrior.org/
brew install speedtest-cli        # https://github.com/sivel/speedtest-cli
brew install duti                 # https://github.com/moretension/duti/
brew install mas                  # https://github.com/mas-cli/mas
brew install dockutil
brew install the_silver_searcher  # https://github.com/ggreer/the_silver_searcher
```

## SSH Key

```
git config --global user.name "Matthias Koch"
git config --global user.email "ithrowexceptions@gmail.com"
ssh-keygen -t rsa -b 4096 -C "ithrowexceptions@gmail.com"
eval "$(ssh-agent -s)"
#touch ~/.ssh/config
ssh-add -K ~/.ssh/id_rsa
pbcopy < ~/.ssh/id_rsa.pub
echo Paste key on https://github.com/settings/keys and https://jetbrains.team/m/Matthias.Koch/git
```

## Browser

```
brew cask install firefox
brew cask install google-chrome
brew cask install brave-browser
brew cask install microsoft-edge

default_browser="com.google.Chrome"
#default_browser = "com.mozilla.firefox"
#default_browser = "com.brave.Browser"
#default_browser = "com.microsoft.edgemac"

duti -s ${default_browser} http all
duti -s ${default_browser} https all
```

## Productivity

```
# Remove all docked apps
dockutil --remove all

# Update computer name
sudo scutil --set ComputerName "dawg"
sudo scutil --set HostName "dawg"
sudo scutil --set LocalHostName "dawg"
sudo defaults write /Library/Preferences/SystemConfiguration/com.apple.smb.server NetBIOSName $(scutil --get LocalHostName)

# Configure Finder
defaults write com.apple.finder CreateDesktop false
defaults write com.apple.finder AppleShowAllFiles YES
defaults write com.apple.finder ShowPathbar -bool true
defaults write com.apple.finder ShowStatusBar -bool true
killall Finder

sudo defaults write /Library/Preferences/com.apple.loginwindow LoginwindowText "l33t!"

brew cask install enpass
brew cask install alfred
brew cask install bartender
brew cask install bettertouchtool
brew cask install avibrazil-rdm
brew cask install ferdi
brew cask install spotify
```

## Development

```
brew cask install jetbrains-toolbox
brew cask install visual-studio-code

# http://seriot.ch/resources/utis_graph/utis_graph.pdf
duti -s com.microsoft.VSCode public.plain-text all
duti -s com.microsoft.VSCode public.unix-executable all
duti -s com.microsoft.VSCode public.data all
duti -s com.microsoft.VSCode .dotsettings all
duti -s com.microsoft.VSCode .targets all
duti -s com.microsoft.VSCode .props all
duti -s com.microsoft.VSCode .dotsettings all
duti -s com.microsoft.VSCode .cs all
duti -s com.microsoft.VSCode .csproj all

code --install-extension ms-dotnettools.csharp
code --install-extension cssho.vscode-svgviewer

brew cask install dotnet-sdk
echo 'export PATH=$HOME/.dotnet/tools:$PATH' >> ~/.zshrc
dotnet tool install nuke.globaltool --global
dotnet tool install installsdkglobaltool --global
dotnet tool install dnt --global

brew cask install docker
brew install mono

brew tap adoptopenjdk/openjdk
brew cask install adoptopenjdk8
```

## Repositories

```
git clone git@github.com:matkoch/resharper-plugins ~/code/resharper-plugins
git clone git@github.com:matkoch/matkoch.github.io ~/code/blog
git clone git@github.com:matkoch/thumbnail-generator ~/code/thumbnail-generator
git clone git@github.com:nuke-build/nuke ~/code/nuke
cd ~/code/nuke;              nuke generate-global-solution
cd ~/code/resharper-plugins; nuke generate-global-solution
```

## Other Tools

```
# mas install 975937182 # Fantastical - Calendar & Tasks
mas install 488764545    # The Clock
mas install 1056643111   # Clocker
mas install 1444383602   # GoodNotes 5

brew cask install fantastical
brew cask install dropbox
brew cask install taskwarrior-pomodoro
brew cask install grammarly
brew cask install tunnelblick
brew cask install tripmode
brew cask install royal-tsx

brew cask install camtasia
brew cask install snagit
brew cask install gimp
brew cask install vlc
brew install imagemagick

brew cask install slack
brew cask install skype
brew cask install telegram

brew cask install obs
brew cask install obs-virtualcam
brew cask install loopback
brew cask install streamlabs-obs
```



- https://gist.github.com/squarism/ae3613daf5c01a98ba3a
- https://medium.com/@gveloper/using-iterm2s-built-in-integration-with-tmux-d5d0ef55ec30
- mas install 1254981365 # Contrast
- https://zellwk.com/blog/mac-setup-2/
- echo "Reboot, hold ⌘+R, open terminal and type: csrutil disable"
