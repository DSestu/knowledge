# Windows context menu managers.

[Doc](https://learn.microsoft.com/en-us/windows/win32/shell/context-menu-handlers?redirectedfrom=MSDN)

Classes are either user or administrator.

`*` being a specific class.

* User: `HKEY_CURRENT_USER\Software\Classes\*\shell`

* Admin: `HKEY_CLASSES_ROOT\*\shell`

Common classes:

* `directory` (`Directory` for admin): Right click on a directory

* `directory\Background` (`Directory\Background` for admin): Right click on a directory background

* On a file: `*`

* `Drive`: mounted drive



The registry format is the same for each type of context:

1. create a key with a given name (it will not appear in the context).

* The value of the key (`(Default)`) is the name in the context manager.

* Optional: `Icon`: `REG_EXPAND_SZ`: Path to the icon, can be an exe. You can use `(...),2` if you want the second icon of the exe.

* Optional: Display only on shift click: Create a empty string key `Extended`.

* Optional: Change menu entry location: add a string value named `Position` with one of: `Top`, `Bottom`

2. Create a new key named `command`

* Change the `(Default)` value to the command, `%1` being the path you right clicked on. eg. `myprogrampath\path\path\executable.exe "%1"`
