# Basic setup

1. Change windows version and activate it

Can be done with the following:

[https://github.com/massgravel/Microsoft-Activation-Scripts](https://github.com/massgravel/Microsoft-Activation-Scripts)

Or with the powershell command

```powershell
irm https://massgrave.dev/get | iex
```

2. Make sure to update everything via windows update

3. Update drivers using CCleaner Pro (activated standalones are easily findable)

**Either**

4. Install AtlasOS

[https://atlasos.net/](https://atlasos.net/)

**Or remove bloatware + various tweaks**

[https://github.com/Raphire/Win11Debloat](https://github.com/Raphire/Win11Debloat)

In Powershell:

```powershell
& ([scriptblock]::Create((irm "https://debloat.raphi.re/")))
```

## Various Tweaks

[https://github.com/ChrisTitusTech/winutil](https://github.com/ChrisTitusTech/winutil)

```powershell
irm "https://christitus.com/win" | iex
```

## WSL

Set WSL to see the same interfaces as Windows

> C:/Users/{user}/.wslconfig

```
[wsl2]
localhostforwarding=true
networkingMode=mirrored
```
