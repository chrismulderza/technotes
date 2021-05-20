# Setting up a developer workstation on Ubuntu 21.04

## Software Installation

### Install common software tools

First install some common unix/linux tools.

```bash
$ sudo apt install curl apt-transport-https tmux default-jdk default-jdk-doc maven ubuntu-restricted-addons ubuntu-restricted-extras gnome-sushi
```

### Microsoft Tools (Teams, VSCode)

I prefer to use the Microsoft provided Debian repositories, so that Teams is updated along with the other packages on the machine

Add the Microsoft repo gpg key:

```bash
$ curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
$ sudo install -o root -g root -m 644 packages.microsoft.gpg /etc/apt/trusted.gpg.d/
$ rm -f packages.microsoft.gpg
```

Create a repositories in `/etc/apt/sources.list.d`:

```bash
$ sudo tee /etc/apt/sources.list.d/ms-teams.list << 'EOL'
deb [arch=amd64] https://packages.microsoft.com/repos/ms-teams stable main
EOL

$ sudo tee /etc/apt/sources.list.d/ms-vscode.list << 'EOL'
deb [arch=amd64,arm64,armhf signed-by=/etc/apt/trusted.gpg.d/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main
EOL
```

Update APT repositories and install:

```bash
$ sudo apt update
$ sudo apt install teams code
```

After installation, remove the extra APT sources created by the Teams and VSCode debs:

```bash
$ sudo rm /etc/apt/sources.list.d/vscode.list 
$ sudo rm /etc/apt/sources.list.d/teams.list 
```

## Configure Tools

### Git

Setup global git username details:

```bash
$ git config --global user.name "John Doe"
$ git config --global user.name "jdoe@example.com"
```

### Podman and friends

Configure BTRFS graph driver globally and for the user:

```bash
$ sudo tee /etc/containers/storage.conf << 'EOL'
[storage]

driver = "btrfs"
EOL
```

```bash
$ mkdir ~/.config/containers
$ tee /home/cmulder/.config/containers/storage.conf << 'EOL'
[storage]

driver = "btrfs"
EOL
```


