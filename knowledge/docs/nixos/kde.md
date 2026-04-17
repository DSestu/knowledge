# KDE Plasma

Minimal config to get a fully working Plasma 6 desktop, then the options you will actually tweak — fonts, HiDPI, Wayland, keyboard layout, and the declarative route for shortcuts, themes, and panels via `plasma-manager`.

## In this page

- [Minimal working Plasma 6](#minimal-working-plasma-6)
- [X11 vs Wayland](#x11-vs-wayland)
- [HiDPI / fractional scaling](#hidpi-fractional-scaling)
- [Fonts](#fonts)
- [Keyboard layout](#keyboard-layout)
- [Essentials on top of Plasma](#essentials-on-top-of-plasma)
- [Useful programs modules](#useful-programs-modules)
- [KWin tweaks that matter](#kwin-tweaks-that-matter)
- [Declarative Plasma config (plasma-manager)](#declarative-plasma-config-plasma-manager)
- [Desktop widgets and plasmoids](#desktop-widgets-and-plasmoids)
- [Suspend, hibernate, lock](#suspend-hibernate-lock)
- [Flatpak integration on Wayland](#flatpak-integration-on-wayland)

## Minimal working Plasma 6

```nix
{ pkgs, ... }:
{
  services.xserver.enable = true;
  services.desktopManager.plasma6.enable = true;
  services.displayManager.sddm.enable = true;
  services.displayManager.sddm.wayland.enable = true;

  # PipeWire for audio (KDE expects it)
  services.pipewire = {
    enable = true;
    alsa.enable = true;
    alsa.support32Bit = true;
    pulse.enable = true;
    jack.enable = true;
  };
  services.pulseaudio.enable = false;
  security.rtkit.enable = true;

  # XDG portal — needed for screen sharing, file pickers, Flatpak integration on Wayland
  xdg.portal = {
    enable = true;
    extraPortals = [ pkgs.kdePackages.xdg-desktop-portal-kde ];
  };
}
```

That is a bootable KDE. Everything below is refinement.

## X11 vs Wayland

SDDM supports both. Users pick the session on the login screen.

```nix
services.displayManager.sddm.wayland.enable = true;   # enable Wayland session
services.xserver.enable = true;                       # still needed for Xwayland + X11 fallback
```

> Nvidia + Wayland: works with recent drivers but enable `hardware.nvidia.modesetting.enable = true;` and use a non-ancient `nvidiaSettings`. See the GPU snippet in [snippets.md](snippets.md).

## HiDPI / fractional scaling

Plasma 6 supports fractional scaling natively in its Display settings. No config file needed — pick the scale in System Settings → Display.

For tty / early-boot HiDPI:

```nix
console.font = "Lat2-Terminus16";
boot.loader.systemd-boot.consoleMode = "max";  # larger boot menu text
```

## Fonts

```nix
fonts = {
  enableDefaultPackages = true;
  packages = with pkgs; [
    noto-fonts
    noto-fonts-emoji
    noto-fonts-cjk-sans
    liberation_ttf
    fira-code
    fira-code-symbols
    jetbrains-mono
    font-awesome
    nerd-fonts.fira-code
    nerd-fonts.jetbrains-mono
  ];
  fontconfig.defaultFonts = {
    serif     = [ "Noto Serif" ];
    sansSerif = [ "Noto Sans" ];
    monospace = [ "JetBrains Mono" ];
    emoji     = [ "Noto Color Emoji" ];
  };
};
```

## Keyboard layout

X11 session:

```nix
services.xserver.xkb = {
  layout = "fr";
  variant = "";
  options = "ctrl:nocaps";   # caps lock → control
};
```

Virtual console (tty):

```nix
console.keyMap = "fr";
```

## Essentials on top of Plasma

```nix
environment.systemPackages = with pkgs.kdePackages; [
  kate
  kcalc
  kcolorchooser
  kolourpaint
  ksystemlog
  partitionmanager
  filelight          # disk-usage visualizer
  isoimagewriter     # USB writer with checksum verification
];

environment.systemPackages = with pkgs; [
  wl-clipboard       # Wayland clipboard CLI
  kdePackages.qtstyleplugin-kvantum  # theming engine for Qt
];
```

## Useful programs modules

```nix
programs.kdeconnect.enable = true;    # phone ↔ desktop sync, opens firewall ports
programs.partition-manager.enable = true;
programs.kclock.enable = true;
programs.dconf.enable = true;         # needed by some GTK apps (VSCode theme, etc.)
```

## KWin tweaks that matter

Most KWin settings live in `~/.config/kwinrc` and are edited via System Settings, not `configuration.nix`. A couple of exceptions:

```nix
# Disable baloo file indexer system-wide if you find it annoying
services.kdeconnect.enable = true;

# Enable touchpad gestures / libinput tuning
services.libinput = {
  enable = true;
  touchpad = {
    naturalScrolling = true;
    tapping = true;
    disableWhileTyping = true;
  };
};
```

> For anything per-user (themes, wallpapers, panel layouts, keyboard shortcuts), use **home-manager** with the `plasma-manager` module — see the next section.

## Declarative Plasma config (plasma-manager)

Plasma's settings live in `~/.config/kglobalshortcutsrc`, `~/.config/kwinrc`, `~/.config/plasmarc`, etc. You can override any of them declaratively via [`plasma-manager`](https://github.com/nix-community/plasma-manager), a home-manager module. Keyboard shortcuts, panels, wallpapers, themes, cursor, icons, and arbitrary config-file keys are all supported.

### Wire plasma-manager into your flake

```nix
# flake.nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    home-manager.url = "github:nix-community/home-manager";
    home-manager.inputs.nixpkgs.follows = "nixpkgs";
    plasma-manager.url = "github:nix-community/plasma-manager";
    plasma-manager.inputs.nixpkgs.follows = "nixpkgs";
    plasma-manager.inputs.home-manager.follows = "home-manager";
  };

  outputs = { nixpkgs, home-manager, plasma-manager, ... }: {
    nixosConfigurations.myhost = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        ./configuration.nix
        home-manager.nixosModules.home-manager
        {
          home-manager.sharedModules = [ plasma-manager.homeManagerModules.plasma-manager ];
          home-manager.users.alice = import ./home.nix;
        }
      ];
    };
  };
}
```

### Shortcut overrides — the ones from [linux/kde.md](../linux/kde.md)

```nix
# home.nix
{
  programs.plasma = {
    enable = true;

    shortcuts = {
      # Virtual desktops: ctrl+win+arrow to switch
      "kwin"."Switch One Desktop to the Left"  = "Meta+Ctrl+Left";
      "kwin"."Switch One Desktop to the Right" = "Meta+Ctrl+Right";
      "kwin"."Switch One Desktop Up"           = "Meta+Ctrl+Up";
      "kwin"."Switch One Desktop Down"         = "Meta+Ctrl+Down";

      # Move current window to other desktop
      "kwin"."Window One Desktop to the Left"  = "Meta+Ctrl+Shift+Left";
      "kwin"."Window One Desktop to the Right" = "Meta+Ctrl+Shift+Right";
      "kwin"."Window One Desktop Up"           = "Meta+Ctrl+Shift+Up";
      "kwin"."Window One Desktop Down"         = "Meta+Ctrl+Shift+Down";

      # Views
      "kwin"."ShowDesktopGrid"                 = "Meta+Tab";
      "kwin"."Overview"                        = "Meta+W";

      # Window stickiness
      "kwin"."Window On All Desktops"          = "Meta+X";

      # Clear a default you don't want (set to empty list)
      "kwin"."Walk Through Windows"            = [ ];
    };
  };
}
```

> Each key is `"<component>"."<action>"`. Component names are lowercase (`kwin`, `plasmashell`, `kmix`). Action names match what you see in **System Settings → Shortcuts**. If unsure, set the shortcut once via the GUI, then `cat ~/.config/kglobalshortcutsrc` to see the exact `Component` + `Action` strings to copy.

### Custom shortcut → shell command

```nix
programs.plasma.hotkeys.commands."launch-yazi" = {
  name = "Launch Yazi in Konsole";
  key = "Meta+E";
  command = "konsole -e yazi";
};
```

### Theme, icons, cursor, wallpaper

```nix
programs.plasma.workspace = {
  theme       = "breeze-dark";
  colorScheme = "BreezeDark";
  cursor = { theme = "Breeze_Snow"; size = 24; };
  iconTheme   = "Papirus-Dark";
  wallpaper   = /home/alice/Pictures/wall.jpg;
  # or: wallpaperSlideShow = { path = /home/alice/Pictures/walls; interval = 600; };
};

programs.plasma.fonts = {
  general  = { family = "Noto Sans";      pointSize = 10; };
  fixedWidth = { family = "JetBrains Mono"; pointSize = 10; };
};
```

### KWin behavior (effects, virtual desktops count)

```nix
programs.plasma.kwin = {
  virtualDesktops = {
    number = 4;
    rows   = 1;
    names  = [ "main" "web" "chat" "scratch" ];
  };
  effects = {
    blur.enable           = true;
    minimization.animation = "magiclamp";
    desktopSwitching.animation = "slide";
  };
  nightLight = {
    enable = true;
    mode   = "times";
    temperature.night = 3200;
    time = { evening = "20:00"; morning = "06:00"; };
  };
};
```

### Panel layout

```nix
programs.plasma.panels = [{
  location = "bottom";
  height = 40;
  widgets = [
    "org.kde.plasma.kickoff"
    "org.kde.plasma.pager"
    "org.kde.plasma.icontasks"
    "org.kde.plasma.marginsseparator"
    "org.kde.plasma.systemtray"
    "org.kde.plasma.digitalclock"
  ];
}];
```

### Overriding any rc-file key (escape hatch)

When plasma-manager doesn't expose a setting, you can still write directly to any KDE config file via its `configFile` attr. Keys use the same `FileName -> GroupName -> KeyName = value` structure as the rc files:

```nix
programs.plasma.configFile = {
  "kwinrc" = {
    "Windows" = {
      "FocusPolicy" = "ClickToFocus";
      "DelayFocusInterval" = 300;
    };
    "Compositing" = {
      "OpenGLIsUnsafe" = false;
      "Backend" = "OpenGL";
    };
  };
  "kdeglobals" = {
    "KDE"."SingleClick" = false;
  };
};
```

This covers **any** setting Plasma stores — if you can change it in System Settings, you can pin it here.

### Workflow tip

To discover the exact key names for any setting:

1. Change the setting via System Settings (GUI).
2. `diff` your `~/.config/{kwinrc,kglobalshortcutsrc,plasmarc,kdeglobals}` against a clean state (`git` your `~/.config` in a throwaway repo, or compare to a fresh user account).
3. Copy the changed `[Section] Key=value` into `programs.plasma.configFile` or the appropriate typed attr.

## Desktop widgets and plasmoids

Plasma widgets (also called **plasmoids**) live in two places:

- **Panel widgets** — in the taskbar/dock. Clock, system tray, task switcher, notifications, media controls.
- **Desktop widgets** — floating on the desktop background. Folder view, weather, comic strip, system monitor, sticky notes.

### Adding a widget via the GUI

- **Panel:** right-click the panel → *Add Widgets*, or Edit Panel mode (right-click → *Enter Edit Mode*) and drag from the widget picker.
- **Desktop:** right-click empty desktop → *Add Widgets*.

The dialog lists every installed widget. Drag to a panel or drop on the desktop. Right-click a placed widget → *Configure* for its options. Hover + drag the handle (appears when edit mode is on) to move or resize.

### Getting more widgets from the KDE Store

In the *Add Widgets* dialog, click **Get New Widgets…** to browse [store.kde.org](https://store.kde.org) directly. Installs land in `~/.local/share/plasma/plasmoids/` — per-user, no root needed. This works on NixOS exactly like on any other distro.

Install a `.plasmoid` archive manually:

```bash
# install
kpackagetool6 -t Plasma/Applet -i ./my-widget.plasmoid

# list installed
kpackagetool6 -t Plasma/Applet -l

# remove
kpackagetool6 -t Plasma/Applet -r org.kde.plasma.my-widget
```

### Widgets packaged in nixpkgs

Some widgets are proper NixOS packages — system-wide install, survives reinstalls. Search:

```bash
nix search nixpkgs 'plasma.*applet'
nix search nixpkgs 'kdePackages.plasma'
```

Install examples:

```nix
environment.systemPackages = with pkgs.kdePackages; [
  plasma-systemmonitor     # modern system monitor
  plasma-workspace         # ships many built-in plasmoids
  plasma-browser-integration
];
```

Not every widget from `store.kde.org` is packaged. For those, fall back to the GUI install path — its files live in your home directory and are thus either backed up by impermanence (if you persist `~/.local/share/plasma/plasmoids`) or declared in home-manager via `xdg.dataFile`.

### Declarative panel widgets (plasma-manager)

Panel widget layout is fully supported — each entry in the `widgets` list is the full applet name:

```nix
programs.plasma.panels = [{
  location = "bottom";
  height = 40;
  widgets = [
    "org.kde.plasma.kickoff"              # app launcher
    "org.kde.plasma.pager"                # virtual desktops
    "org.kde.plasma.icontasks"            # task switcher
    "org.kde.plasma.marginsseparator"
    "org.kde.plasma.systemtray"
    "org.kde.plasma.digitalclock"
    "org.kde.plasma.showdesktop"
  ];
}];
```

### Declarative desktop widgets (harder, escape-hatch)

Desktop widget placement is trickier because Plasma stores it in "containments" (one per screen/activity) with positional geometry. Plasma-manager's first-class support is partial. The practical approach:

1. Set up the layout once via the GUI — add widgets, position, configure.
2. `cat ~/.config/plasma-org.kde.plasma.desktop-appletsrc` to see what Plasma wrote.
3. Paste the relevant sections into `programs.plasma.configFile.<rcfile>`:

```nix
programs.plasma.configFile."plasma-org.kde.plasma.desktop-appletsrc" = {
  "Containments"."1"."plugin" = "org.kde.plasma.folder";
  "Containments"."1"."Applets"."2"."plugin" = "org.kde.plasma.systemmonitor";
  # ... position, geometry, per-applet config copied from the GUI-generated file
};
```

Messier than themes or shortcuts, but it locks the layout in git.

### Useful built-ins (no install needed)

- **Folder View** — behave-like-a-folder desktop (classic KDE 3 feel).
- **System Monitor** — CPU / RAM / network / disk graphs, inline on desktop or in the panel.
- **Comic** — daily webcomic strip on the desktop.
- **Weather Report** — current conditions + forecast.
- **Sticky Notes** — scratchpad, supports markdown.
- **Clipboard** — history picker (`Meta+V`).
- **Media Controller** — MPRIS2 controls for whatever's playing.
- **Pager** — tiny view of your virtual desktops.
- **Activities** — switch between Plasma "activities" (sets of windows / wallpaper / widgets).

All of these are already in your *Add Widgets* dialog after enabling Plasma 6.

## Suspend, hibernate, lock

```nix
services.logind = {
  lidSwitch = "suspend";
  lidSwitchExternalPower = "ignore";
  extraConfig = ''
    HandlePowerKey=suspend
    IdleAction=suspend
    IdleActionSec=15min
  '';
};
```

Hibernate needs a swap that is ≥ RAM and `boot.resumeDevice` set. See the swap snippet in [snippets.md](snippets.md).

## Flatpak integration on Wayland

```nix
services.flatpak.enable = true;
```

After a rebuild, add Flathub manually (one-time):

```bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

Flatpak apps will now open files, share screens, and themes correctly via the KDE portal.

## Next

- [dev-tools.md](dev-tools.md) — developer tooling layered on top of KDE.
- [snippets.md](snippets.md) — GPU, Bluetooth, printing, power management.
