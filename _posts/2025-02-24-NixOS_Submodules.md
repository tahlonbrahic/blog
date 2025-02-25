---
title: "NixOS Submodules: The Documentation You Could Never Find..."
date: 2025-02-24
---

'''nix
{
  options.mod = mkOption {
    description = "submodule example";
    type = with types; submodule {
      options = {
        foo = mkOption {
          type = int;
        };
        bar = mkOption {
          type = str;
        };
      };
    };
  };
}
'''
