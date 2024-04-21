# nixos-chromebook

A method of setting up audio, keyd, etc on NixOS.

> [!NOTE]
> I am not affiliated with Chrultrabook. Please don't ask them for help regarding NixOS.

## Prerequisites

- Working internet connection
- A [supported](https://docs.chrultrabook.com/docs/firmware/supported-devices.html) Chromebook
- An understanding of Linux command line
- Patience

### Installation/Configuration

#### Step 1: Create Configuration file(s)

> [!TIP]
> Note that the configurations are applied to `configuration.nix` located in `/etc/nixos`.

> [!TIP]
> Running the command `sudo nixos-rebuild switch` will rebuild your installation and apply the changes made.

First things first, for Chromebook-related configuration, we will create `chrome-device.nix`.

```bash
sudo touch /etc/nixos/chrome-device.nix
```

Then at the top of our `configuration.nix` we import `chrome-device.nix`.

```nix
# configuration.nix
# Edit this configuration file to define what should be installed on
# your system.  Help is available in the configuration.nix(5) man page
# and in the NixOS manual (accessible by running ‘nixos-help’).

{ config, pkgs, ... }:

{
  imports =
    [
      # Include the results of the hardware scan.
      ./hardware-configuration.nix
      ./chrome-device.nix
    ];

    # the rest of your configuration...
}

```

#### Step 2: ALSA UCM configuration

Build the package in `chrome-device.nix` and setup Keyd

```nix
# chrome-device.nix
{ config, pkgs, lib, ... }:

let
  cb-ucm-conf = with pkgs; alsa-ucm-conf.overrideAttrs {
    wttsrc = fetchFromGitHub {
      owner = "WeirdTreeThing";
      repo = "alsa-ucm-conf-cros";
      rev = "6b395ae73ac63407d8a9892fe1290f191eb0315b";
      hash = "sha256-GHrK85DmiYF6FhEJlYJWy6aP9wtHFKkTohqt114TluI=";
    };
    unpackPhase = ''
      runHook preUnpack
      tar xf "$src"
      runHook postUnpack
    '';

    installPhase = ''
      runHook preInstall
      mkdir -p $out/share/alsa
      cp -r alsa-ucm*/ucm2 $out/share/alsa
      runHook postInstall
    '';
  }; 
in
{
    services.keyd = {
    enable = true;
    keyboards.internal = {
      ids = [
        "k:0001:0001"
        "k:18d1:5044"
        "k:18d1:5052"
        "k:0000:0000"
        "k:18d1:5050"
        "k:18d1:504c"
        "k:18d1:503c"
        "k:18d1:5030"
        "k:18d1:503d"
        "k:18d1:505b"
        "k:18d1:5057"
        "k:18d1:502b"
        "k:18d1:5061"
      ];
      settings = {
        main = {
          f1 = "back";
          f2 = "forward";
          f3 = "refresh";
          f4 = "f11";
          f5 = "scale";
          f6 = "brightnessdown";
          f7 = "brightnessup";
          f8 = "mute";
          f9 = "volumedown";
          f10 = "volumeup";
          back = "back";
          forward = "forward";
          refresh = "refresh";
          zoom = "f11";
          scale = "scale";
          brightnessdown = "brightnessdown";
          brightnessup = "brightnessup";
          mute = "mute";
          volumedown = "volumedown";
          volumeup = "volumeup";
          sleep = "coffee";
        };
        meta = {
          f1 = "f1";
          f2 = "f2";
          f3 = "f3";
          f4 = "f4";
          f5 = "f5";
          f6 = "f6";
          f7 = "f7";
          f8 = "f8";
          f9 = "f9";
          f10 = "f10";
          back = "f1";
          forward = "f2";
          refresh = "f3";
          zoom = "f4";
          scale = "f5";
          brightnessdown = "f6";
          brightnessup = "f7";
          mute = "f8";
          volumedown = "f9";
          volumeup = "f10";
          sleep = "f12";
        };
        alt = {
          backspace = "delete";
          meta = "capslock";
          brightnessdown = "kbdillumdown";
          brightnessup = "kbdillumup";
          f6 = "kbdillumdown";
          f7 = "kbdillumup";
        };
        control = {
          f5 = "print";
          scale = "print";
        };
        controlalt = {
          backspace = "C-A-delete";
        };
      };
    };
  };

  # add your audio setup modprobes here

  environment = {
    systemPackages = [ pkgs.sof-firmware ];
    sessionVariables.ALSA_CONFIG_UCM2 = "${cb-ucm-conf}/share/alsa/ucm2";
    # AUDIO SETUP FOR < 23.11 AND UNSTABLE
  };

  # AUDIO SETUP FOR > 24.05

  system.replaceRuntimeDependencies = [
    ({
      original = pkgs.alsa-ucm-conf;
      replacement = cb-ucm-conf;
    })
  ];
}

```

The rest of this process varies between AVS and SOF. Pay attention to the `# AUDIO SETUP FOR...` comment(for your NixOS version), this will be replaced later in configuration.

You can check your NixOS channel with:

```bash
nix-channel --list
```

CPU generations to determine AVS or SOF for your Chromebook

| AVS             | SOF             |
| --------------- | --------------- |
| - Skylake       | - Alderlake     |
| - Kabylake      |  - Jasperlake   |
| - Apollolake    |  - Tigerlake    |
|                 |  - Cometlake    |
|                 |  - Geminilake   |
|                 |  - Braswell     |
|                 |  - Baytrail     |

If your CPU generation isn't listed, please make a pull request!