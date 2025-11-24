# Tools

# Enable 2 finger support for Chrome (forward backward)

Change shortcuts accordingly

```
(...chrome...) --enable-features=TouchpadOverscrollHistoryNavigation
```

# My EZA setup

Zsh config
`~/.zshrc`

```
alias l='eza -Bhm --icons --no-user --git --time-style long-iso --group-directories-first --color=always --color-scale=age -F --no-permissions -s extension --git-ignore'
alias la='l -a'
alias ll='l -la'
alias lt='ll -T'
```

My EZA theme:
`~/.config/eza/theme.yml`

```yml
colourful: true

filekinds:
  normal: { foreground: "#c0caf5" }
  directory: { foreground: "#7aa2f7" }
  symlink: { foreground: "#2ac3de" }
  pipe: { foreground: "#414868" }
  block_device: { foreground: "#e0af68" }
  char_device: { foreground: "#e0af68" }
  socket: { foreground: "#414868" }
  special: { foreground: "#9d7cd8" }
  executable: { foreground: "#42a5f5" }
  mount_point: { foreground: "#a3be8c" }

perms:
  user_read: { foreground: "#2ac3de" }
  user_write: { foreground: "#bb9af7" }
  user_execute_file: { foreground: "#9ece6a" }
  user_execute_other: { foreground: "#9ece6a" }
  group_read: { foreground: "#2ac3de" }
  group_write: { foreground: "#ff9e64" }
  group_execute: { foreground: "#9ece6a" }
  other_read: { foreground: "#2ac3de" }
  other_write: { foreground: "#ff007c" }
  other_execute: { foreground: "#9ece6a" }
  special_user_file: { foreground: "#ff007c" }
  special_other: { foreground: "#db4b4b" }
  attribute: { foreground: "#737aa2" }

size:
  major: { foreground: "#2ac3de" }
  minor: { foreground: "#9d7cd8" }
  number_byte: { foreground: "#a9b1d6" }
  number_kilo: { foreground: "#89ddff" }
  number_mega: { foreground: "#2ac3de" }
  number_giga: { foreground: "#ff9e64" }
  number_huge: { foreground: "#ff007c" }
  unit_byte: { foreground: "#a9b1d6" }
  unit_kilo: { foreground: "#89ddff" }
  unit_mega: { foreground: "#2ac3de" }
  unit_giga: { foreground: "#ff9e64" }
  unit_huge: { foreground: "#ff007c" }

users:
  user_you: { foreground: "#3d59a1" }
  user_root: { foreground: "#bb9af7" }
  user_other: { foreground: "#2ac3de" }
  group_yours: { foreground: "#89ddff" }
  group_root: { foreground: "#bb9af7" }
  group_other: { foreground: "#c0caf5" }

links:
  normal: { foreground: "#89ddff" }
  multi_link_file: { foreground: "#2ac3de" }

git:
  new: { foreground: "#9ece6a" }
  modified: { foreground: "#bb9af7" }
  deleted: { foreground: "#db4b4b" }
  renamed: { foreground: "#2ac3de" }
  typechange: { foreground: "#2ac3de" }
  ignored: { foreground: "#545c7e" }
  conflicted: { foreground: "#ff9e64" }

git_repo:
  branch_main: { foreground: "#737aa2" }
  branch_other: { foreground: "#b4f9f8" }
  git_clean: { foreground: "#292e42" }
  git_dirty: { foreground: "#bb9af7" }

security_context:
  colon: { foreground: "#545c7e" }
  user: { foreground: "#737aa2" }
  role: { foreground: "#2ac3de" }
  typ: { foreground: "#3d59a1" }
  range: { foreground: "#9d7cd8" }

file_type:
  image: { foreground: "#89ddff" }
  video: { foreground: "#b4f9f8" }
  music: { foreground: "#73daca" }
  lossless: { foreground: "#41a6b5" }
  crypto: { foreground: "#db4b4b" }
  document: { foreground: "#a9b1d6" }
  compressed: { foreground: "#ff9e64" }
  temp: { foreground: "#737aa2" }
  compiled: { foreground: "#ffb86c" }
  build: { foreground: "#1abc9c" }
  nix: { foreground: "#7dcfff" }
  python_env: { foreground: "#25cbd3" }

filenames:
  __init__.py: {filename: {foreground: "#ffffff", bold: false, italic: true}, icon: {style: {foreground: "#ffffff", bold: true}}}
  requirements.txt: {filename: {foreground: "#bb9af7"}, icon: {style: {foreground: "#bb9af7"}}}
  pyproject.toml: {filename: {foreground: "#f7768e", bold: true}, icon: {style: {foreground: "#f7768e", bold: true}}}
  poetry.lock: {filename: {foreground: "#89ddff"}, icon: {style: {foreground: "#89ddff"}}}
  setup.py: {filename: {foreground: "#f7768e"}, icon: {style: {foreground: "#f7768e"}}}
  shell.nix: {filename: {foreground: "#7dcfff"}, icon: {style: {foreground: "#7dcfff"}}}
  default.nix: {filename: {foreground: "#7dcfff"}, icon: {style: {foreground: "#7dcfff"}}}
  .env: {filename: {foreground: "#545c7e", italic: true}, icon: {style: {foreground: "#545c7e"}}}
  .venv: {filename: {foreground: "#545c7e", italic: true}, icon: {style: {foreground: "#545c7e"}}}
  .gitignore: {filename: {foreground: "#545c7e"}, icon: {style: {foreground: "#545c7e"}}}
  test_*.py: {filename: {foreground: "#e0af68", italic: true}, icon: {style: {foreground: "#e0af68"}}}

extensions:
  ipynb: {filename: {foreground: "#FD5B5B"}, icon: {style: {foreground: "#FD5B5B"}}}
  py:   {filename: {foreground: "#63A1FF"}, icon: {glyph: "", style: {foreground: "#63A1FF"}}}
  pyc:  {filename: {foreground: "#fdc2be"}, icon: {glyph: "", style: {foreground: "#fdc2be"}}}
  pyo:  {filename: {foreground: "#FFA500"}, icon: {glyph: "", style: {foreground: "#FFA500"}}}
  pyi:  {filename: {foreground: "#FFD700"}, icon: {glyph: "", style: {foreground: "#FFD700"}}}
  toml: {filename: {foreground: "#6FC3DF", bold: true}, icon: {glyph: "", style: {foreground: "#6FC3DF"}}}
  nix:  {filename: {foreground: "#6dcf93"}, icon: {glyph: "", style: {foreground: "#6dcf93"}}}
  yaml: {filename: {foreground: "#ffe066"}, icon: {glyph: "", style: {foreground: "#ffe066"}}}
  yml: {filename: {foreground: "#ffe066"}, icon: {glyph: "", style: {foreground: "#ffe066"}}}
  lock: {filename: {foreground: "#545c7e", italic: true}, icon: {glyph: "", style: {foreground: "#545c7e"}}}
  sh:   {filename: {foreground: "#5af78e", bold: true}, icon: {glyph: "", style: {foreground: "#5af78e"}}}
  json: {filename: {foreground: "#21d7ff"}, icon: {glyph: "", style: {foreground: "#21d7ff"}}}
  md:   {filename: {foreground: "#ffffff", bold: true}, icon: {glyph: "", style: {foreground: "#ffffff"}}}
  test: {filename: {foreground: "#fb61d7", italic: true}, icon: {glyph: "", style: {foreground: "#fb61d7", italic: true}}}
  sql:  {filename: {foreground: "#f6ae2d"}, icon: {glyph: "", style: {foreground: "#f6ae2d"}}}
  csv: {filename: {foreground: "#ffd966"}, icon: {glyph: "", style: {foreground: "#ffd966"}}}
  parquet: {filename: {foreground: "#FF4F4F"}, icon: {glyph: "", style: {foreground: "#FF4F4F"}}}


punctuation: { foreground: "#414868" }
date: { foreground: "#424242" }
inode: { foreground: "#737aa2" }
blocks: { foreground: "#737aa2" }
header: { foreground: "#a9b1d6" }
octal: { foreground: "#ff9e64" }
flags: { foreground: "#9d7cd8" }

symlink_path: { foreground: "#89ddff" }
control_char: { foreground: "#ff9e64" }
broken_symlink: { foreground: "#ff007c" }
broken_path_overlay: { foreground: "#ff007c" }
```

# Debian, restart trackpad service

```bash
sudo modprobe -r psmouse && sudo modprobe psmouse
```

# Execute arbitrary commands when file change

<https://eradman.com/entrproject/>

Do something when a file in a subfolder changes

The `-c` arg clears the screen everytime.

```bash
find . -type f -print0 | xargs -0 ls -d | entr -rc git status
```

# Disk usage analyzer

> **[baobab](https://doc.ubuntu-fr.org/baobab)**

# Terminal multiplexer

> **[byobu](https://www.byobu.org/)** (really nice)

Make sure to enable mouse support and keybinds:

* Shift + F12

* Ctrl + F12

# CLI file explorer

> [ranger](https://github.com/ranger/ranger)

![Cheat sheet](https://ranger.github.io/cheatsheet.png)
