---
title: "NixOS Impermanence Part 2: The NixOS Module"
date: 2025-02-26 20:00:00 -400
categories: ["nix" "security"]
---

One of the

Here is a **summarized** version of the configuration
from the NixOS Impermanence module:

```nix
_: {
  config = {
    systemd.services = {
      "persist-${escapeSystemdPath targetFile}" = {
        description = "Bind mount or link ${targetFile} to ${mountPoint}";
        wantedBy = ["local-fs.target"];
        before = ["local-fs.target"];
        path = [pkgs.util-linux];
        unitConfig.DefaultDependencies = false;
        serviceConfig = {
          Type = "oneshot";
          RemainAfterExit = true;
          ExecStart = "${mountFile} ${mountPoint} ${targetFile} ${escapeShellArg enableDebugging}";
          ExecStop = pkgs.writeShellScript "unbindOrUnlink-${escapeSystemdPath targetFile}" ''
            set -eu
            if [[ -L ${mountPoint} ]]; then
                rm ${mountPoint}
            else
                umount ${mountPoint}
                rm ${mountPoint}
            fi
          '';
        };
      };
    };

    fileSystems = mkIf (directories != []) bindMounts;

    system.activationScripts = {
      "createPersistentStorageDirs" = {
        deps = ["users" "groups"];
        text = "${dirCreationScript}";
      };
      "persist-files" = {
        deps = ["createPersistentStorageDirs"];
        text = "${persistFileScript}";
      };
    };

    boot.initrd = {
      systemd.services = mkIf config.boot.initrd.systemd.enable {
        create-needed-for-boot-dirs = {
          wantedBy = ["initrd-root-device.target"];
          requires = deviceUnits;
          after = deviceUnits;
          before = ["sysroot.mount"];
          serviceConfig.Type = "oneshot";
          unitConfig.DefaultDependencies = false;
          script = createNeededForBootDirs;
        };
      };
      postResumeCommands =
        mkIf (!config.boot.initrd.systemd.enable)
        (mkAfter createNeededForBootDirs);
    };
  };
}
```

```nix

```

```nix

```
