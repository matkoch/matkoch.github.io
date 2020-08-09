# Setup macOS environment

## Homebrew

```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

## Terminal

```
brew install git
brew install svn
brew install mc
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
brew install ack

# https://github.com/Homebrew/homebrew-cask-fonts/tree/master/Casks
brew tap homebrew/cask-fonts
brew cask install font-roboto
brew cask install font-cascadia
brew cask install font-fira-code
brew cask install font-source-code-pro
brew cask install font-fira-code-nerd-font
brew cask install font-meslo-lg-nerd-font

# https://github.com/romkatv/powerlevel10k
brew cask install iterm2
/bin/bash -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
brew install romkatv/powerlevel10k/powerlevel10k
echo 'source /usr/local/opt/powerlevel10k/powerlevel10k.zsh-theme' >>! ~/.zshrc
p10k configure
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

default_browser="com.mozilla.firefox"
#default_browser = "com.google.Chrome"
#default_browser = "com.brave.Browser"
#default_browser = "com.microsoft.edgemac"

#duti -s ${default_browser} public.html all
#duti -s ${default_browser} public.xhtml all
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

# Configure environment
defaults write com.apple.finder CreateDesktop false
defaults write com.apple.finder AppleShowAllFiles YES
defaults write com.apple.finder ShowPathbar -bool true
defaults write com.apple.finder ShowStatusBar -bool true
defaults write com.apple.finder FXPreferredViewStyle -string "Nlsv"

defaults write com.apple.AppleMultitouchTrackpad Clicking -bool true
defaults write com.apple.driver.AppleBluetoothMultitouch.trackpad Clicking -bool true
defaults write com.apple.menuextra.battery ShowPercent -string "YES"
#defaults write com.apple.menuextra.battery ShowTime -string "YES"
defaults write com.apple.dock mru-spaces -bool false # Spaces rearrange
defaults write NSGlobalDomain AppleShowAllExtensions -bool true
defaults write com.apple.finder FXEnableExtensionChangeWarning -bool false
defaults write com.apple.Finder FXPreferredViewStyle -string "clmv"                         # Nlsv, icnv, clmv, Flwv
defaults write com.apple.finder NewWindowTargetPath -string "PfHm"                          # https://www.jamf.com/jamf-nation/discussions/30370/solved-setting-finder-to-open-new-windows-to-onedrive-folder-via-script
#defaults write /Library/Preferences/com.apple.security GKAutoRearm -bool false             # https://www.defaults-write.com/disable-gatekeeper-on-your-mac/#more-1305
# find / -name ".DS_Store" -exec rm {} \;
killall Finder

sudo defaults write /Library/Preferences/com.apple.loginwindow LoginwindowText "l33t!"

brew cask install enpass
brew cask install 1password
brew cask install dropbox
brew cask install marta
brew cask install alfred
brew cask install bartender
brew cask install bettertouchtool
brew cask install avibrazil-rdm
brew cask install ferdi
brew cask install spotify
brew cask install fantastical

brew tap homebrew/cask-drivers
brew cask install logitech-options
```

## Development

```
brew cask install jetbrains-toolbox
brew cask install visual-studio-code

# $(osascript -e 'id of app "Visual Studio Code"')
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
duti -s com.microsoft.VSCode .nuspec all
duti -s com.microsoft.VSCode .json all
duti -s com.microsoft.VSCode .xml all
duti -s com.microsoft.VSCode .yaml all
duti -s com.microsoft.VSCode .yml all
duti -s com.microsoft.VSCode .sh all
duti -s com.microsoft.VSCode .css all
duti -s com.microsoft.VSCode .js all
duti -s com.microsoft.VSCode .kt all
duti -s com.microsoft.VSCode .java all
duti -s com.apple.archiveutility .nupkg all

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
ssh-add -K ~/.ssh/id_rsa

git clone git@github.com:matkoch/resharper-plugins ~/code/resharper-plugins
git clone git@github.com:matkoch/matkoch.github.io ~/code/blog
git clone git@github.com:matkoch/thumbnail-generator ~/code/thumbnail-generator
git clone git@github.com:matkoch/nuke ~/code/nuke
git clone git@github.com:matkoch/ferdi-youtrack ~/Library/Application\ Support/Ferdi/recipes/dev/youtrack
git clone git@github.com:matkoch/ferdi-jetbrains-space ~/Library/Application\ Support/Ferdi/recipes/dev/jetbrains-space

cd ~/code/nuke
git remote add upstream git@github.com:nuke-build/nuke
nuke generate-global-solution

cd ~/code/resharper-plugins
nuke generate-global-solution
```

## Other Tools

```
mas install 975937182    # Fantastical - Calendar & Tasks
mas install 488764545    # The Clock
mas install 1056643111   # Clocker
mas install 1444383602   # GoodNotes 5
mas install 933627574    # Silicio 3 for Spotify + iTunes

brew cask install taskwarrior-pomodoro
brew cask install grammarly
brew cask install tunnelblick
brew cask install tripmode
brew cask install royal-tsx

brew cask install camtasia
brew cask install snagit
brew cask install gifox
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
- https://apple.stackexchange.com/questions/382098/how-to-enable-tap-to-click-using-keyboard-only
- http://www.defaults-write.com/
- https://marketmix.com/de/mac-osx-umfassende-liste-der-terminal-defaults-commands/
