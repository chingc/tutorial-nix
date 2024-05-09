# Nix

Hello, this is a record of my journey learning [Nix](https://nixos.org).

I'm writing this mainly as a quickstart and quick reference for my future self, but I hope others find it helpful.

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
go version go1.22.3 darwin/arm64

$ node --version
v20.12.2

$ python -V
Python 3.11.9

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

With Nix, we can declare all dependencies explicitly, and produce a script that will always run on any machine that supports Nix.

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
#! nix shell nixpkgs/e2dd4e18cc1c7314e24154331bae07df76eb582f#python312
#! nix --ignore-environment --command python

print("n", "n^2")
for n in range(1, 10):
    print(n, n * n)
```

In this example we've included a git commit hash of the Nixpkgs repository. This ensures that the script will always run with the exact same package versions, everywhere.

## Declarative Shell Environments

We can create a file that defines an environment. This can be shared with anyone to recreate the same environment on a different machine.

Create a `flake.nix` file:

```
{
  description = "A fun shell environment";

  # Pin to a specific commit for reproducibility
  inputs.nixpkgs.url = "nixpkgs/e2dd4e18cc1c7314e24154331bae07df76eb582f";

  inputs.flake-utils.url = "github:numtide/flake-utils";

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

We've already seen a few examples of version pinning by using the git commit hash of the Nixpkgs repository. There are several places where you can find these hashes.

Go to [status.nixos.org](https://status.nixos.org) for the latest commits to the official Nix channels.

If you are looking for an older version of a program that isn't in the current channels, that could be a little tricky.

For example, the package `nginx` currently points to v1.26.0 in the unstable channel and v1.24.0 in the 23.11 channel. However, if you need the commit hash that contains nginx v1.22.1, then the easiest place to find it is from an unofficial source called [Nix Package Versions](https://lazamar.co.uk/nix-versions/).

The alternative is to look for it in [Hydra](https://hydra.nixos.org), the NixOS build system. This method can be slow, difficult, and unsuccessful. Check out this [forum thread](https://discourse.nixos.org/t/steps-to-find-a-commit-with-cached-version-of-package/5674) to get an overview on how to navigate Hydra.

I'd like to hear from you if you have better suggestions on how to lookup commits for older packages.

With that out of the way, let's recap how to version pin.

tbd

## Search for Packages

Go to <https://search.nixos.org/packages> or use the command line.

```bash
$ nix search nixpkgs neovim
```

#### Wrapped vs Unwrapped

The key difference between wrapped and unwrapped packages in NixOS is that wrapped packages are configured to work seamlessly within the NixOS environment, while unwrapped packages are the raw, unmodified versions.

In most cases, users should install the wrapped version of a package, as it is preconfigured to work correctly on NixOS. The unwrapped version is primarily used when further customization or overriding of the package is required, as it serves as the base for creating a new wrapped derivation with additional modifications.

## References

Below are some of the main sources I've used.

- [GitHub: DeterminateSystems/nix-installer](https://github.com/DeterminateSystems/nix-installer)
- [GitHub: NixOS/nix](https://github.com/NixOS/nix)
- [GitHub: NixOS/nixpkgs](https://github.com/NixOS/nixpkgs)
- [Nix Reference Manual: nix search](https://nix.dev/manual/nix/latest/command-ref/new-cli/nix3-search)
- [Nix Reference Manual: nix shell](https://nix.dev/manual/nix/latest/command-ref/new-cli/nix3-shell)
- [Nix Reference Manual: nix run](https://nix.dev/manual/nix/latest/command-ref/new-cli/nix3-run)
- [Nix Package Versions](https://lazamar.co.uk/nix-versions/)
- [NixOS Hydra](https://hydra.nixos.org)
- [NixOS Search Packages](https://search.nixos.org/packages)
- [NixOS Status](https://status.nixos.org)
- [nix.dev](https://nix.dev)
- [Zero to Nix](https://zero-to-nix.com)

Other notable sources:

- [Kubukoz Blog: Nix Flakes First Steps](https://blog.kubukoz.com/flakes-first-steps/)

Additional resources:

- <https://nixos.org/learn>
