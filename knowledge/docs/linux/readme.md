# Tools

# Execute arbitrary commands when file change

https://eradman.com/entrproject/

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

