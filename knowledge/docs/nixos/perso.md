# Perso

Home manager setup from Debian

## Enable flakes & install home-manager

```bash
mkdir -p ~/.config/nix
echo 'experimental-features = nix-command flakes' >> ~/.config/nix/nix.conf
nix flake update nixpkgs
nix-channel --add https://github.com/nix-community/home-manager/archive/master.tar.gz home-manager
nix-channel --update
nix-shell '<home-manager>' -A install
```nix run home-manager/master -- init --switch


## Initialize home-manager

```bash
nix run home-manager/master -- init --switch
```
