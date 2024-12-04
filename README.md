# README: Nix

Hello, this is a record of my journey learning [Nix](https://nixos.org).

I'm writing this mainly as a quickstart and reference for my future self, but I hope others find it helpful.

This is not comprehensive. It will just go over the basics and some of the more useful features.

Note: This doc will be using the newer `nix` cli interface and flakes.

## What is it?

Nix is a purely functional package manager. It helps to avoid the installation and versioning pain points of software development.

That doesn't sound like much. However...

Suppose you need golang, node, and python. You can create a shell environment that has those tools with a single command.

```bash
$ go; node; python
zsh: command not found: go
zsh: command not found: node
zsh: command not found: python

$ nix shell nixpkgs#go nixpkgs#nodejs nixpkgs#python3

$ go version
go version go1.23.3 darwin/arm64

$ node --version
v20.18.0

$ python --version
Python 3.12.7

$ exit

$ go; node; python
zsh: command not found: go
zsh: command not found: node
zsh: command not found: python
```

We were able to get access to golang, node, and python instantly. Actually, the initial run takes a few moments, but subsequent runs are fast!

We skipped the setup docs, we didn't edit our shell profile, and we didn't have to worry about `PATH` or other environment variables.

Best of all, we didn't pollute the user environment. Everything is gone once you `exit` or `Ctrl-D` the shell.

Other benefits?

Nix builds packages in isolation from each other. This ensures that they are reproducible and don't have undeclared dependencies, so if a package works on one machine, it will also work on another.

Nix makes it trivial to share development and build environments for your projects, regardless of what programming languages and tools you're using.

## Install

The easiest way to install Nix (Linux, macOS, WSL2) is to use [The Determinate Nix Installer](https://zero-to-nix.com/concepts/nix-installer).

It installs Nix with flake support and the unified CLI feature already enabled. It also stores a receipt for the install process to allow for a clean uninstall.

Note: Directions to upgrade or uninstall can be found in their [GitHub repo](https://github.com/DeterminateSystems/nix-installer).

## Run Programs Directly

We've already seen how a shell environment can be created with `nix shell`.

With `nix run` we can skip creating the shell and run programs directly.

```bash
$ nix run nixpkgs#cowsay Hello, Nix!

$ nix run nixpkgs#lolcat -- --help

$ nix run nixpkgs#fortune | nix run nixpkgs#lolcat
```

## Reproducible Scripts

A trivial script with non-trivial dependencies.

```bash
#! /bin/bash

curl https://github.com/NixOS/nixpkgs/releases.atom | xml2json | jq .
```

This script fetches XML content from a URL, converts it to JSON, and formats it for better readability.

It requires curl, xml2json, jq, and bash. If any of these dependencies are not present on the system running the script, it will fail partially or altogether.

With Nix, we can declare all dependencies explicitly, and produce a script that will always run on any machine that has Nix.

```bash
#! /usr/bin/env nix
#! nix shell nixpkgs#bash nixpkgs#curl nixpkgs#jq nixpkgs#python312Packages.xmljson
#! nix --ignore-environment --command bash

curl https://github.com/NixOS/nixpkgs/releases.atom | xml2json | jq .
```

- The `--ignore-environment` option prevents the script from implicitly using programs that may already exist on the system.
- The `--command` option specifies the interpreter that will be invoked by nix shell after it has obtained the dependencies and initialized the environment.

Notice how `nix shell` was only specified once when using multiple nix shell shebangs.

As long as the system has Nix, scripts can be written in any language without worrying about dependencies.

```bash
#! /usr/bin/env nix
#! nix shell nixpkgs/nixos-24.11#python3
#! nix --ignore-environment --command python

print("n", "n^2")
for n in range(1, 10):
    print(n, n * n)
```

In this example we've specified the `nixos-24.11` stable channel. This helps reproducibility. You can even use a git commit hash of the Nixpkgs repository for further granularity.

## Declarative Shell Environments

We can create a file that defines an environment. This can be shared with anyone to recreate the same environment on a different machine.

Create a `flake.nix` file:

```
{
    description = "A fun shell environment";

    inputs = {
        nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
        flake-utils.url = "github:numtide/flake-utils";
    };

    outputs = { self, nixpkgs, flake-utils }:
        flake-utils.lib.eachDefaultSystem (system:
            let
                pkgs = import nixpkgs { inherit system; config = {}; overlays = []; };
            in {
                devShells.default = pkgs.mkShellNoCC {

                    # Packages to include
                    packages = with pkgs; [
                        cowsay
                        lolcat
                        python3Full
                    ];

                    # Define environment variables
                    env = {
                        GREETING = "Hello, Nix!";
                    };

                    # Run a shell script on environment startup
                    shellHook = ''
                        echo $GREETING | cowsay | lolcat
                    '';

                };
            }
        );
}
```

Enter the environment by running `nix develop` in the same directory as `flake.nix`.

Note: If the directory you're working in is a git repository you may get an error indicating `flake.nix` doesn't exist. Stage the file with `git add` to get nix to recognize it. I don't know. It's weird like that.

If you make changes to `flake.nix` just `exit` or `Ctrl-D` to exit the environment and restart it with `nix develop`.

## More on Versioning

Without pinning or locking your tools and dependencies to specific versions you'll eventually hit the point of development where a bug is affecting others but somehow it "works on my machine". It's often very unpleasant and very difficult to debug.

We've seen from an earlier example how to pin to a specific Nix channel. The channels can be found at [status.nixos.org](https://status.nixos.org).

Tip: [Which channel branch should I use?](https://nix.dev/concepts/faq#which-channel-branch-should-i-use)

## Search for Packages

Go to <https://search.nixos.org/packages> or use the command line.

```bash
$ nix search nixpkgs neovim
```

#### Wrapped vs Unwrapped

The key difference between wrapped and unwrapped packages in NixOS is that wrapped packages are configured to work seamlessly within the NixOS environment, while unwrapped packages are the raw, unmodified versions.

In most cases, users should install the wrapped version of a package, as it is preconfigured to work correctly on NixOS. The unwrapped version is primarily used when further customization or overriding of the package is required, as it serves as the base for creating a new wrapped derivation with additional modifications.

## Thoughts

tbd

## References

Below are some of the main sources I've used.

- [GitHub: DeterminateSystems/nix-installer](https://github.com/DeterminateSystems/nix-installer)
- [GitHub: NixOS/nix](https://github.com/NixOS/nix)
- [GitHub: NixOS/nixpkgs](https://github.com/NixOS/nixpkgs)
- [Nix Reference Manual: nix run](https://nix.dev/manual/nix/latest/command-ref/new-cli/nix3-run)
- [Nix Reference Manual: nix search](https://nix.dev/manual/nix/latest/command-ref/new-cli/nix3-search)
- [Nix Reference Manual: nix shell](https://nix.dev/manual/nix/latest/command-ref/new-cli/nix3-env-shell)
- [nix.dev](https://nix.dev)
- [nix.dev: Best Practices](https://nix.dev/guides/best-practices)
- [Zero to Nix](https://zero-to-nix.com)

Additional resources:

- [Kubukoz Blog: Nix Flakes First Steps](https://blog.kubukoz.com/flakes-first-steps/)
- [Nix Package Versions](https://lazamar.co.uk/nix-versions/)
- [NixOS Hydra](https://hydra.nixos.org)
- [NixOS Learn](https://nixos.org/learn)
- [NixOS Search Packages](https://search.nixos.org/packages)
- [NixOS Status](https://status.nixos.org)
