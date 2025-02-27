---
title: "NixOS Impermanence Part 1: The NixOS Module Options"
date: 2025-02-26
categories: ["nix" "security"]
---

In my opinion the most interesting part of NixOS is its unique ability to
produce reproducable builds. Yet one of the things that prevents this are
everyday use. Application state, log data, and configurations polute 
various directories in your system after an initial NixOS install and stay
that way unless explicitly managed by something. As NixOS does not include
that something by default the nix-community has built a set of modules that
are meant to supplement much of the configuration that would be needed on the
user-end to fix these issues. This module is called Impermanence.

Now this would not be Nix if there was not a lack of documentation.
The Impermanence project is no difference. As of writing, the only
documentation can be found on it's README and to be concise it, like many
other Nix repositories, can only help make you more confused.

Part of my aim in the Nix community is help bring an understanding to Nix,
it's code, and other parts of its implementation by presenting the underlying
structure of this highly abstracted system. This series will break down 
Impermanence step by step to help the community understand it and implement it.

The first step in this is to condense both modules options and configurations.
This article will cover the first sub-section of that: NixOS Impermance Options.
The below options have been extracted from the NixOS module and summarized
in order to make human evaluation easier.

**Common Options:**

```nix
{
  options = {
    persistentStoragePath = lib.mkOption {
      type = lib.types.path;
      default = config.persistentStoragePath;
      defaultText = "environment.persistence.‹name›.persistentStoragePath";
      description = ''
        The path to persistent storage where the real
        file or directory should be stored.
      '';
    };
    home = lib.mkOption {
      type = lib.types.nullOr lib.types.path;
      default = null;
      internal = true;
      description = ''
        The path to the home directory the file is
        placed within.
      '';
    };
    enableDebugging = lib.types.mkOption {
      type = lib.types.bool;
      default = config.enableDebugging;
      defaultText = "environment.persistence.‹name›.enableDebugging";
      internal = true;
      description = ''
        Enable debug trace output when running
        scripts. You only need to enable this if asked
        to.
      '';
    };
  };
}
```

It is important to note in the above common options that these options are
meant to be declared by the environment.persistence attribute set that
gets defined later on. This is different from how I have seen many other
modules use option declarations where users declare options not through
a proxy by directly e.g. impermanence.enable = true;

These are not the only options.
The options available from the top are as follows.

**Top-Level Options:**

```nix
{lib, ...}: {
  defaultPerms, # variable
  name, # variable
  commonOpts, # variable
  dirOpts, # variable
  fileOpts, # variable
  rootFile, # variable
  rootDir, # variable
}: {
  options = {
    enable = lib.mkOption {
      type = lib.types.bool;
      default = true;
      description = "Whether to enable this persistent storage location.";
    };

    persistentStoragePath = lib.mkOption {
      type = lib.types.path;
      default = name;
      defaultText = "‹name›";
      description = ''
        The path to persistent storage where the real
        files and directories should be stored.
      '';
    };

    users = lib.mkOption {
      type = lib.types.attrsOf (
        lib.types.submodule (
          {
            name,
            config,
            ...
          }: let
            userDefaultPerms = {
              inherit (defaultPerms) mode;
              user = name;
              group = users.${userDefaultPerms.user}.group;
            };
            fileConfig = {config, ...}: {
              parentDirectory = rec {
                directory = dirOf config.file;
                dirPath = lib.concatPaths [config.home directory];
                inherit (config) persistentStoragePath home;
                defaultPerms = userDefaultPerms;
              };
              filePath = lib.concatPaths [config.home config.file];
            };
            userFile = lib.types.submodule [
              commonOpts
              fileOpts
              {inherit (config) home;}
              {
                parentDirectory = lib.mkDefault userDefaultPerms;
              }
              fileConfig
            ];
            dirConfig = {config, ...}: {
              defaultPerms = lib.mkDefault userDefaultPerms;
              dirPath = lib.concatPaths [config.home config.directory];
            };
            userDir = lib.types.submodule ([
                commonOpts
                dirOpts
                {inherit (config) home;}
                dirConfig
              ]
              ++ (lib.mapAttrsToList (n: v: {${n} = lib.mkDefault v;}) userDefaultPerms));
          in {
            options = {
              # Needed because defining fileSystems
              # based on values from users.users
              # results in infinite recursion.
              home = lib.mkOption {
                type = lib.types.path;
                default = "/home/${userDefaultPerms.user}";
                defaultText = "/home/<username>";
                description = ''
                  The user's home directory. Only
                  useful for users with a custom home
                  directory path.

                  Cannot currently be automatically
                  deduced due to a limitation in
                  nixpkgs.
                '';
              };

              files = lib.mkOption {
                type = lib.types.listOf (lib.coercedTo lib.types.str (f: {file = f;}) userFile);
                default = [];
                example = [
                  ".screenrc"
                ];
                description = ''
                  Files that should be stored in
                  persistent storage.
                '';
              };

              directories = lib.mkOption {
                type = lib.types.listOf (lib.coercedTo lib.str (d: {directory = d;}) userDir);
                default = [];
                example = [
                  "Downloads"
                  "Music"
                  "Pictures"
                  "Documents"
                  "Videos"
                ];
                description = ''
                  Directories to bind mount to
                  persistent storage.
                '';
              };
            };
          }
        )
      );
      default = {};
      description = ''
        A set of user submodules listing the files and
        directories to link to their respective user's
        home directories.

        Each attribute name should be the name of the
        user.
      '';
    };

    files = lib.mkOption {
      type = lib.types.listOf (lib.coercedTo lib.types.str (f: {file = f;}) rootFile);
      default = [];
      example = [
        "/etc/machine-id"
        "/etc/nix/id_rsa"
      ];
      description = ''
        Files that should be stored in persistent storage.
      '';
    };

    directories = lib.mkOption {
      type = lib.types.listOf (lib.coercedTo lib.types.str (d: {directory = d;}) rootDir);
      default = [];
      example = [
        "/var/log"
        "/var/lib/bluetooth"
      ];
      description = ''
        Directories to bind mount to persistent storage.
      '';
    };

    hideMounts = lib.mkOption {
      type = lib.types.bool;
      default = false;
      example = true;
      description = ''
        Whether to hide bind mounts from showing up as mounted drives.
      '';
    };

    enableDebugging = lib.mkOption {
      type = lib.types.bool;
      default = false;
      internal = true;
      description = ''
        Enable debug trace output when running
        scripts. You only need to enable this if asked
        to.
      '';
    };

    enableWarnings = lib.mkOption {
      type = lib.types.bool;
      default = true;
      description = ''
        Enable non-critical warnings.
      '';
    };
  };
}
```

This module is also a unique case in the NixOS world where
in most cases it has to be supplemented on the user-side.

As-in, you have some initial setup to do.

```nix

```
