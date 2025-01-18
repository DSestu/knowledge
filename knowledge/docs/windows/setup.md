# Basic setup

1. Change windows version and activate it

Can be done with the following:

(https://github.com/massgravel/Microsoft-Activation-Scripts)[https://github.com/massgravel/Microsoft-Activation-Scripts]

Or with the powershell command

```powershell
irm https://massgrave.dev/get | iex
```
2. Make sure to update everything via windows update

3. Update drivers using CCleaner Pro (activated standalones are easily findable)

4. Install AtlasOS

(https://atlasos.net/)[https://atlasos.net/]

## WSL

Set WSL to see the same interfaces as Windows

> C:/Users/{user}/.wslconfig
```
[wsl2]
localhostforwarding=true
networkingMode=mirrored
```