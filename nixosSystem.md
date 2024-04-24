# lib.nixosSystem

## pkgs

Do NOT set `pkgs` in `lib.nixosSystem`
there's this stupid pattern which comes from some old guide or youtube video which goes like this:

```nix
let
  system = "x86_64-linux";
  pkgs = import nixpkgs {
    inherit system;
    overlays = [];
  };
in
{
  nixosConfigurations."nixos" = lib.nixosSystem {
    inherit system pkgs;
    #           HERE^
    modules = [];
    specialArgs = {
        inherit pkgs;
    #     or HERE^
    };
  };
}
```

There is never a reason to do this and it will stop all `nixpkgs.` options from working SILENTLY WITHOUT ERROR

if you want to instantiate your own nixpkgs instance use the nixpkgs `readOnlyPkgs`
[module](https://github.com/NixOS/nixpkgs/blob/ba10489eae3b2b2f665947b516e7043594a235c8/flake.nix#L61-L72)

## specialArgs

There is no `specialArgs` checking in `lib.nixosSystem`
you can pass `config`,`lib`,`pkgs`, or `modulesPath` (just to name the obvious ones) and completely break your config

There's no reason to random values to `specialArgs` such as:

- `hostname`: can be found under `config.networking.hostName` already
- `user`: just why?
- `stateVersion`: this should be hard-coded per-configuration so there's no use in passing it around

Instead make your own option and access those arbitrary values through `config`.
Give the manual entry on writing modules a read if you haven't already:
https://nixos.org/manual/nixos/unstable/#sec-writing-modules

## system

You don't need to pass `system` to `lib.nixosSystem`
if you view the [source](https://github.com/NixOS/nixpkgs/blob/ba10489eae3b2b2f665947b516e7043594a235c8/nixos/lib/eval-config.nix#L12-L15)
for `lib.nixosSystem` you'll see it straight up says:

> this can be set modularly, it would be nice to remove

and if you use a generated hardware-configuration you already have a line like this:

```nix
nixpkgs.hostPlatform = lib.mkDefault "x86_64-linux";
```

if you don't have that line or are making an abstraction over `lib.nixosSystem`
simply turn

```nix
lib.nixosSystem {
  system = "x86_64-linux";
  modules = [
...
```

into

```nix
lib.nixosSystem {
  modules = [
  {
    nixpkgs.hostPlatform = "x86_64-linux";
  }
```

## Implicit flake registry and NIX_PATH configuration

Ever since [this commit](https://github.com/NixOS/nixpkgs/commit/e456032addae76701eb17e6c03fc515fd78ad74f)
lib.nixosSystem sets nix.path and nix.registry This isn't well documented (like anything nix)
but shouldn't really cause issues...

## Ideal case

IMO this is what an ideal setup looks like (unless you're abstracting host creation):

```nix
lib.nixosSystem {
  modules = [
    # A single module entry point
    ./hosts/nixos
  ];
  # Pass inputs to your configuration for easy use
  specialArgs = { inherit inputs;};
}
```

## it's stupid

It's just a shallow wrapper around `lib.evalModules` and you can easily use `lib.evalModules` directly for example:

```nix
lib.evalModules {
  modules = [
    ./hosts/nixos
    (import "${nixpkgs}/nixos/modules/module-list.nix")
  ];
  specialArgs = {
    inherit inputs;
    modulesPath = "${nixpkgs}/nixos/modules";
  };
};
```

should be identical to the "Ideal case" except for the implicit NIX_PATH and flake registry configuration.

A full example using evalModules to initialize a host would look roughly like
this:

```nix
{
  description = "A very basic flake";

  # Opinionated way of constructing an input
  inputs.nixpkgs = {
    type = "github";
    owner = "NixOS";
    repo = "nixpkgs";
    ref = "nixos-unstable";
  };

  outputs = {
    self,
    nixpkgs,
  } @ inputs: let
    inherit (nixpkgs) lib;
  in {
    nixosConfigurations.bald-frog = lib.evalModules {
      modules = builtins.concatLists [
        (import "${nixpkgs}/nixos/modules/module-list.nix")

        (lib.singleton {
          # having this set to true requires basePath to be set
          # which is done so in nixosSystem while it imports
          # ${nixpkgs}/nixos/lib/eval-config.nix
          documentation.nixos.enable = false;
          nixpkgs.hostPlatform = "x86_64-linux";
        })
      ];

      specialArgs = {
        inherit inputs self;
        modulesPath = "${nixpkgs}/nixos/modules";
      };
    };
  };
}
```
