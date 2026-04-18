# The Nix language in one page

You can run Nix for a while by copy-pasting snippets, but eventually you hit an option whose type is `attribute set of list of package`, or you want to understand why `{ pkgs, ... }: { ... }` appears at the top of every module, or why `with pkgs; [ git vim ]` works but `[ git vim ]` doesn't. At that point the Nix language stops being incidental and starts being the thing you are reading.

This page teaches the subset of the Nix language you actually need to read and write `configuration.nix` and `home.nix` without guessing. It is deliberately practical — no coverage of advanced evaluation semantics or packaging internals. For those, see the [official language tutorial](https://nix.dev/tutorials/nix-language) and [noogle.dev](https://noogle.dev) (searchable library-function index).

## In this page

- [The two shapes you see 95% of the time](#the-two-shapes-you-see-95-of-the-time)
- [Values and their types](#values-and-their-types)
- [Attribute sets](#attribute-sets)
- [Functions](#functions)
- [`let ... in` bindings](#let--in-bindings)
- [`with` — a shortcut, and its footgun](#with-a-shortcut-and-its-footgun)
- [`inherit`](#inherit)
- [`rec` — self-referential attribute sets](#rec-self-referential-attribute-sets)
- [`if ... then ... else`](#if-then-else)
- [Strings and interpolation](#strings-and-interpolation)
- [`import`](#import)
- [`lib` and common helpers](#lib-and-common-helpers)
- [Reading a real module](#reading-a-real-module)

## The two shapes you see 95% of the time

Every Nix file evaluates to a **value**. In practice, the two values you read and write all day are:

- **An attribute set literal** — `{ foo = 1; bar = "hello"; }`. Think "dict" / "object".
- **A function** — `{ pkgs, ... }: { ... }`. Everything left of the `:` is the argument pattern; everything right is the body.

The rule of thumb for reading a `.nix` file: if the first non-comment character is `{` followed by identifiers and a `:`, it's a function. If the first character is `{` followed by `identifier = ...;`, it's a bare attribute set.

```nix
# This file is an attribute set literal:
{
  name = "alice";
  age = 30;
}
```

```nix
# This file is a function of one argument (an attribute set),
# returning an attribute set:
{ pkgs, ... }:
{
  environment.systemPackages = [ pkgs.ripgrep pkgs.fd ];
}
```

Every NixOS or home-manager module is the second shape.

## Values and their types

| Type                 | Example                                | Notes                                                                 |
| -------------------- | -------------------------------------- | --------------------------------------------------------------------- |
| String               | `"hello"`, `''multi\nline''`           | Two string syntaxes; see [Strings](#strings-and-interpolation).       |
| Integer              | `42`                                   | No float suffix needed; Nix has separate floats.                      |
| Float                | `3.14`                                 |                                                                       |
| Boolean              | `true`, `false`                        |                                                                       |
| Null                 | `null`                                 | Distinct from "not set". Some modules use `null` to mean "auto".      |
| Path                 | `./modules/fish.nix`, `/etc/nixos`     | A literal path. Relative paths are relative to the file they're in.   |
| List                 | `[ 1 2 3 ]`                            | Elements are **space-separated**, no commas.                          |
| Attribute set        | `{ a = 1; b = "x"; }`                  | Key-value pairs ending with `;`. Like a dict.                         |
| Function             | `x: x + 1`                             | One argument `x`, body after `:`.                                     |
| Derivation           | `pkgs.hello`                           | A special attribute set produced by `stdenv.mkDerivation` and friends. Evaluates to a build recipe. |

The surprise for most newcomers: **lists have no commas**. `[ "a" "b" "c" ]`, not `[ "a", "b", "c" ]`. A stray comma is a syntax error.

## Attribute sets

The bread and butter. Attribute sets are ordered by name at evaluation time (so print order is deterministic) and accessed with `.`:

```nix
let
  user = {
    name = "alice";
    groups = [ "wheel" "docker" ];
    email = "alice@example.com";
  };
in
  user.name          # "alice"
```

### Nested sets and dotted keys

```nix
{
  services.openssh.enable = true;
  services.openssh.ports = [ 22 2222 ];
}
```

That is sugar for:

```nix
{
  services = {
    openssh = {
      enable = true;
      ports = [ 22 2222 ];
    };
  };
}
```

The module system happily accepts either form. Most real configs mix both: flat dotted keys for single settings, nested blocks when a section gets busy.

### Updating an attribute set

`//` merges two sets (right wins on key conflicts):

```nix
let
  defaults = { port = 22; open = false; };
  overrides = { open = true; };
in
  defaults // overrides
# → { port = 22; open = true; }
```

`//` is shallow — nested keys are replaced wholesale, not merged. For deep merges inside a module, the module system does the right thing automatically (see `lib.mkMerge`).

### Checking and defaulting

- `attrs ? key` — `true` if `key` is present.
- `attrs.key or default` — returns `default` if `key` is absent.

```nix
cfg.port or 22
```

## Functions

Every function in Nix takes exactly one argument. Multi-argument calls are either currying (`x: y: x + y`) or — much more commonly — a single attribute-set argument with a **destructuring pattern**.

### Single-argument

```nix
let
  addOne = x: x + 1;
in
  addOne 41                  # 42
```

Application is whitespace — no parentheses around arguments.

### Attribute-set argument with destructuring

This is the pattern every NixOS/home-manager module uses:

```nix
{ pkgs, lib, config, ... }:
{
  environment.systemPackages = [ pkgs.git ];
}
```

Breaking this down:

- `{ pkgs, lib, config, ... }` — the argument is an attribute set; from it, pull out `pkgs`, `lib`, `config`.
- `...` — accept any other attributes silently (NixOS passes in lots of things: `options`, `specialArgs`, ...). Without `...`, Nix would error on unexpected attributes.
- `:` — starts the function body.
- `{ environment.systemPackages = ...; }` — the returned value, an attribute set.

So the whole file is: "a function that, given an attribute set with at least `pkgs`, `lib`, and `config` in it, returns a module that adds git to `environment.systemPackages`." **That is what a module is.** There is no other definition.

### Default values

```nix
{ enable ? true, port ? 22, ... }:
{ services.openssh = { inherit enable port; }; }
```

## `let ... in` bindings

Give names to intermediate values:

```nix
let
  username = "alice";
  home = "/home/${username}";
  packages = with pkgs; [ git fish ];
in
{
  users.users.${username} = {
    isNormalUser = true;
    home = home;
    packages = packages;
  };
}
```

Bindings are **recursive within the `let`** — later bindings can reference earlier ones, and (carefully) vice versa. Bindings are **lazy**, so order inside the `let` does not matter the way it would in a strict language.

## `with` — a shortcut, and its footgun

`with attrs; expr` brings every attribute of `attrs` into scope for `expr`:

```nix
# Without with
environment.systemPackages = [ pkgs.git pkgs.vim pkgs.htop ];

# With with
environment.systemPackages = with pkgs; [ git vim htop ];
```

Very common at the edges of a list, and idiomatic. The footgun: **`with` at module top level can shadow names in confusing ways**. Rule of thumb — use `with` inside a list or a small expression, almost never at the top of a module.

```nix
# Bad — name shadowing trap
with pkgs;
{
  environment.systemPackages = [ git ];
  # Now `lib.*` refers to pkgs.lib, not the module system's lib. Surprise bugs.
}

# Good — local scope only
{
  environment.systemPackages = with pkgs; [ git vim htop ];
}
```

## `inherit`

Shorthand for "set a key to a value of the same name from the enclosing scope":

```nix
let
  name = "alice";
  email = "alice@example.com";
in
{
  user = { inherit name email; };
  # equivalent to:
  # user = { name = name; email = email; };
}
```

`inherit (source) a b c;` pulls names from a specific attribute set:

```nix
{ config, pkgs, ... }:
let
  inherit (pkgs) git vim htop;
  # same as: git = pkgs.git; vim = pkgs.vim; htop = pkgs.htop;
in
{
  environment.systemPackages = [ git vim htop ];
}
```

Ubiquitous in real configs.

## `rec` — self-referential attribute sets

Normally attribute-set keys can't see each other; `rec { ... }` makes them recursive:

```nix
rec {
  username = "alice";
  homeDir  = "/home/${username}";      # works — username is in scope
  sshKey   = "${homeDir}/.ssh/id_ed25519";
}
```

Without `rec`, the second and third lines would be evaluation errors.

`rec` has its own footgun: shadowing any key in scope with the `rec`'s own keys can cause surprise. In modules, `let ... in { ... }` is usually cleaner than `rec { ... }`.

## `if ... then ... else`

Expression-style, always requires `else`:

```nix
{ config, lib, ... }:
{
  services.openssh.enable = if config.networking.hostName == "laptop" then false else true;
}
```

For module-level conditionals, prefer `lib.mkIf`:

```nix
{
  services.openssh.enable = lib.mkIf config.myServices.remoteAdmin true;
}
```

`lib.mkIf` defers evaluation and interacts cleanly with the module system's merging logic.

## Strings and interpolation

Two string syntaxes:

### Double-quoted

```nix
"hello ${username}"
"path is /etc/nixos/${name}.nix"
```

Escapes: `\"`, `\\`, `\n`, `\t`, `\${` (escaped dollar-brace).

### Double-single-quote (indented string)

```nix
''
  #!/usr/bin/env bash
  echo "Hello ${username}"
''
```

Indentation is auto-stripped to the common prefix — perfect for embedding scripts, config files, or multi-line text. Escapes: `''${` (escaped interpolation), `'''` (literal triple-quote).

Use indented strings whenever the content itself is multi-line; they read much better than `"line1\nline2\nline3"`.

## `import`

`import path` evaluates the file at `path` and returns its value. If that value is a function (most modules are), you apply it:

```nix
let
  users = import ./users.nix;                   # users.nix returns an attrset of users
  fishModule = import ./modules/fish.nix;       # a function { pkgs, ... }: ...
in
{
  # Import is how you assemble modules, though `imports = [ ./file.nix ];` in the module system
  # does this automatically for you — no explicit import call needed.
}
```

In day-to-day `configuration.nix`, `imports = [ ./modules/fish.nix ];` is the idiom; `import` shows up when you need the returned value directly.

## `lib` and common helpers

`lib` (`pkgs.lib`, or the first argument to most modules) is Nix's standard library. A handful of functions appear constantly:

| Helper                | What it does                                                   |
| --------------------- | -------------------------------------------------------------- |
| `lib.mkIf cond value` | Apply `value` only if `cond` is `true` (module-system aware).  |
| `lib.mkDefault value` | Set a default that other modules can override trivially.       |
| `lib.mkForce value`   | Set a value that wins over other definitions.                  |
| `lib.mkMerge [ a b ]` | Deep-merge multiple module fragments.                          |
| `lib.optional c v`    | `[ v ]` if `c` else `[]`. Common inside package lists.         |
| `lib.optionals c vs`  | `vs` if `c` else `[]`. Plural form.                            |
| `lib.getName pkg`     | Extract a package's pname (useful in `allowUnfreePredicate`).  |
| `lib.versionAtLeast`  | Semver comparisons, e.g. `lib.versionAtLeast pkgs.nix.version "2.18"`. |

Find any library function at [noogle.dev](https://noogle.dev) — searchable, with live examples.

## Reading a real module

Putting it all together — here is a small but real module, and what every line means:

```nix
{ config, pkgs, lib, ... }:

let
  cfg = config.myModules.devShell;
in
{
  options.myModules.devShell = {
    enable = lib.mkEnableOption "my custom dev shell";

    extraPackages = lib.mkOption {
      type        = with lib.types; listOf package;
      default     = [];
      description = "Additional packages to include in the dev shell.";
    };
  };

  config = lib.mkIf cfg.enable {
    environment.systemPackages = with pkgs;
      [ ripgrep fd bat eza ] ++ cfg.extraPackages;

    programs.direnv.enable = true;
  };
}
```

Line by line:

- `{ config, pkgs, lib, ... }:` — a function of one attribute-set argument, destructuring `config`, `pkgs`, `lib`; ignore anything else.
- `let cfg = config.myModules.devShell; in ...` — give the local short name `cfg` to the subtree of `config` this module cares about. Idiomatic in every non-trivial module.
- `options.myModules.devShell = { ... };` — declare that this module contributes two new options. `lib.mkEnableOption` is a one-liner for "a boolean option named `enable`".
- `lib.mkOption { type; default; description; }` — a user-facing option with documentation. `lib.types.listOf package` is the type combinator — "a list of packages".
- `config = lib.mkIf cfg.enable { ... };` — actual configuration this module applies, conditional on `cfg.enable`. `mkIf` makes the module composable with overrides elsewhere.
- `environment.systemPackages = with pkgs; [ ripgrep fd bat eza ] ++ cfg.extraPackages;` — our packages, merged with whatever the user passed. `++` is list concatenation.
- `programs.direnv.enable = true;` — delegate to another module.

Every line uses one feature from this page. Modules are not magic; they are functions returning attribute sets, wired together by the module system.

## Next

- [configuration.md](configuration.md) — the module system, `nixos-rebuild`, option discovery.
- [multi-host-flake.md](multi-host-flake.md) — structuring a flake for multiple machines once the basics feel natural.
- [packages.md](packages.md) — writing derivations (for when you need to package something yourself).
- [noogle.dev](https://noogle.dev) — searchable library-function index.
- [nix.dev](https://nix.dev) — the canonical deeper tutorial.
