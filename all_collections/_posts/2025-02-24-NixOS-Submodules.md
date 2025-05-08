---
title: "NixOS Submodules: The Documentation You Could Never Find..."
date: 2025-02-24 19:00:00 -500
categories: ["nix"]
---

When defining custom options for a NixOS module you may encounter yourself
needing to use deeply nested options. When this becomes the case you have
to decide between using nested attribute sets to declare module options
are using the much more complex and undocumented submodule system.

The following example shows a similar configuration in two different
styles. First is the nested attribute style followed by the
submodule sytle.

```nix
options = {
    kosei.ssh = {
      enable = lib.mkEnableOption "Enable SSH";
      level = lib.mkOption {
        type = lib.types.ints.between 0 2;
        default = 2;
        description = "The SSH access level (0-2).";
      };
      publicKeys = lib.mkOption {
        type = lib.types.listOf lib.types.str;
        default = [];
        description = "List of public SSH keys for authentication.";
      };
    };
```

```nix
options = {
    kosei.ssh = lib.mkOption {
      type = lib.types.submodule {
        options = {
          enable = lib.mkEnableOption "Enable SSH";
          level = lib.mkOption {
            type = lib.types.ints.between 0 2;
            default = 2;
            description = "The SSH access level (0-2).";
          };
          publicKeys = lib.mkOption {
            type = lib.types.listOf lib.types.str;
            default = [];
            description = "List of public SSH keys for authentication.";
          };
        };
      };
    };
```

A violent example of submodule usage can been seen in this
snippet of the nginx server for its NixOS module:

```nix
upstreams = mkOption {
        type = types.attrsOf (types.submodule {
          options = {
            servers = mkOption {
              type = types.attrsOf (types.submodule {
                freeformType = types.attrsOf (types.oneOf [ types.bool types.int types.str ]);
                options = {
                  backup = mkOption {
                    type = types.bool;
                    default = false;
                    description = ''
                      Marks the server as a backup server. It will be passed
                      requests when the primary servers are unavailable.
                    '';
                  };
                };
              });
              description = ''
                Defines the address and other parameters of the upstream servers.
                See [the documentation](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server)
                for the available parameters.
              '';
              default = {};
              example = lib.literalMD "see [](#opt-services.nginx.upstreams)";
            };
            extraConfig = mkOption {
              type = types.lines;
              default = "";
              description = ''
                These lines go to the end of the upstream verbatim.
              '';
            };
          };
        });
        description = ''
          Defines a group of servers to use as proxy target.
        '';
        default = {};
        example = {
          "backend" = {
            servers = {
              "backend1.example.com:8080" = { weight = 5; };
              "backend2.example.com" = { max_fails = 3; fail_timeout = "30s"; };
              "backend3.example.com" = {};
              "backup1.example.com" = { backup = true; };
              "backup2.example.com" = { backup = true; };
            };
            extraConfig = ''
              keepalive 16;
            '';
          };
          "memcached" = {
            servers."unix:/run/memcached/memcached.sock" = {};
          };
        };
      };
```

You could then **import** this module and define
an option like:

```nix
services.nginx.upstreams.<name>.servers.<name>.backup = true
```

#### You should prefer nested attribute sets when

    - You're dealing with simpler configurations.

    - You want everything contained in a single scope.

#### You should prefer Nix submodules when

    - You need modularization and reusability.

    - You expect your configurations to grow.

    - You need need to separate concerns for clarity and maintainability.
