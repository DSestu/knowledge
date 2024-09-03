# Setup

## Using UV as a package manager

[https://docs.astral.sh/uv/getting-started/installation/](https://docs.astral.sh/uv/getting-started/installation/)

Installation instructions can also be found in the [uv](https://github.com/astral-sh/uv/releases) repo.

### Basic commands

Setup a new project with specific python version.

```bash
uv init --python 3.12
```

Then create the virtual environment.

```bash
uv venv
```

You will need to activate the virtual environment.

```bash
. .venv/bin/activate
```

Synchronize the project with the latest version of the dependencies.

```bash
uv sync
```

Add a new dependency to the project.

```bash
uv add requests
```

Add a dependency with extras, eg alternative to `pip install fastapi[standard]`.

```bash
uv add fastapi --extra standard
```

Run a script with updated dependencies.

```bash
uv run python -m my_script
```

## Using devenv/direnv to automatically activate the virtual environment

[Install devenv](https://devenv.sh/getting-started/#installation)

In short:

```bash
sh <(curl -L https://nixos.org/nix/install) --no-daemon

nix-env -iA devenv -f https://github.com/NixOS/nixpkgs/tarball/nixpkgs-unstable

nix-env -i direnv
```

[Install direnv](https://direnv.net/docs/installation.html).

### Initialise the project

```bash
devenv init
```

### Using more recent versions of packages

> You need to change the `devenv.yaml` in order to access to more recent versions of packages *(this is required for the `uv` package)*.

**devenv.yaml**

```yaml
inputs:
  nixpkgs:
    url: github:NixOS/nixpkgs/nixpkgs-unstable
  # pinned terraform to 1.5.7, because post 1.6 it's marked 'unfree' (BSL)
  nixpkgs-for-terraform:
    url: github:NixOS/nixpkgs?rev=4ab8a3de296914f3b631121e9ce3884f1d34e1e5
```

### Tuning the devenv shell

A good starting snippet for `devenv.nix` that will automatically activate the virtual environment.

```nix
{ pkgs, config, inputs,... }:
let
  # pinned terraform, because post 1.6 it's marked 'unfree' (BSL)
  pinned-terraform-nixpkgs = import inputs.nixpkgs-for-terraform { system = pkgs.stdenv.system; };
in
{
  packages = with pkgs; [
    colima # to provide a docker container runtimes
    docker # for docker commands
    uv
  ];

  dotenv.disableHint = true;

  scripts.init.exec = ''
    unset NIX_CC
  '';

  # scripts.start.exec = "uvicorn app.main:app";
  scripts.tests.exec = "pytest tests";
  scripts.itests.exec = "pytest tests_integration";

  # unset PYTHONPATH is necessary to ensure that libraries solely from the virtual environment are used
  enterShell = ''
    unset PYTHONPATH
    . ./.venv/bin/activate
    uv sync
  '';
}
```

The file above will also ensure support for docker images, but first you will need to run:

```bash
colima start
```

## Docker

By using the `devenv`/`direnv` setup above, you will be able to run docker commands without having it installed system-wide.

You will need to make sure that colima is running with the following command:

```bash
colima status
```

If it is not running, you can start it with the following command:

```bash
colima start
```

### Starting snippet to use docker with uv

The following `Dockerfile` will install the latest version of uv and run the `docker_start.sh` script at the root of the project.

```dockerfile
FROM python:3.12-slim-bookworm

# The installer requires curl (and certificates) to download the release archive
RUN apt-get update && apt-get install -y --no-install-recommends curl ca-certificates

# Download the latest installer
ADD https://astral.sh/uv/install.sh /uv-installer.sh

# Run the installer then remove it
RUN sh /uv-installer.sh && rm /uv-installer.sh

# Ensure the installed binary is on the `PATH`
ENV PATH="/root/.cargo/bin/:$PATH"

WORKDIR /environment

COPY . /environment/
```

`docker-compose.yml`

```yaml	
version: '3.8'

services:
  app:
    build: .
    ports:
      - 8008:8008
    volumes:
      - .:/environment
    environment:
      - PYTHONUNBUFFERED=1
    command: sh -c "chmod +x ./docker_start.sh && ./docker_start.sh"
```

Run the following command to start the container.

```bash
docker compose up -d
```

## Docker GUI without Docker desktop

[Portainer community edition](https://docs.portainer.io/) is a nice tool to have a docker GUI without the neeed of Docker Desktop.

Installation instructions [can be found here](https://docs.portainer.io/start/install-ce/server/docker/linux).

To use it, you need only need to run the following command.

```bash
docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:2.21.0
```

Then you can access it at [https://localhost:9443](https://localhost:9443).
