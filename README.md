# NixOS + VSCode + clangd Setup

Quick guide for getting clangd working in VSCode on NixOS for C++ projects.

## The Problem

On NixOS, clangd can't find C++ standard library headers by default because:
1. Clang and GCC libraries are in isolated nix store paths
2. clangd doesn't automatically know about the wrapped compiler's include paths
3. You need to explicitly tell clangd where everything is

## The Solution

### 1. Project Files

Create these files in your project root:

**flake.nix**
```nix
{
  description = "C++ development environment with clangd";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-utils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flake-utils }:
    flake-utils.lib.eachDefaultSystem (system:
      let
        pkgs = nixpkgs.legacyPackages.${system};
      in
      {
        devShells.default = pkgs.mkShell {
          packages = with pkgs; [
            clang
            clang-tools  # includes clangd
            gcc  # for standard library headers
          ];

          shellHook = ''
            echo "C++ dev environment loaded"
            echo "clangd: $(clangd --version | head -n1)"
          '';
        };
      }
    );
}
```

**.envrc**
```bash
use flake
watch_file flake.nix
watch_file flake.lock
```

### 2. Get the Nix Store Paths

After running `direnv allow`, run this to find ALL your paths:
```bash
clang++ -E -x c++ - -v < /dev/null 2>&1 | grep -A 20 "search starts here"
```

This will show the exact include search order. You need ALL five paths:
1. **compiler-rt path**: `/nix/store/XXX-compiler-rt-libc-21.1.2-dev/include`
2. **GCC C++ headers**: `/nix/store/XXX-gcc-14.3.0/include/c++/14.3.0`
3. **GCC platform headers**: `/nix/store/XXX-gcc-14.3.0/include/c++/14.3.0/x86_64-unknown-linux-gnu`
4. **clang resource path**: `/nix/store/XXX-clang-wrapper-21.1.2/resource-root/include`
5. **glibc-dev path**: `/nix/store/XXX-glibc-2.40-66-dev/include`

### 3. Configure clangd

Create `.clangd` with YOUR paths **in this exact order**:

```yaml
CompileFlags:
  Add: 
    - -std=c++17
    - -isystem
    - /nix/store/YOUR-compiler-rt-libc-21.1.2-dev/include
    - -isystem
    - /nix/store/YOUR-gcc-14.3.0/include/c++/14.3.0
    - -isystem
    - /nix/store/YOUR-gcc-14.3.0/include/c++/14.3.0/x86_64-unknown-linux-gnu
    - -isystem
    - /nix/store/YOUR-clang-wrapper-21.1.2/resource-root/include
    - -isystem
    - /nix/store/YOUR-glibc-2.40-66-dev/include

Diagnostics:
  UnusedIncludes: Strict

InlayHints:
  Enabled: Yes
  ParameterNames: Yes
  DeducedTypes: Yes
```

**IMPORTANT**: The order matters! This must match the order from the `clang++` command output.

### 4. VSCode Settings

Create `.vscode/settings.json`:

```json
{
  "C_Cpp.intelliSenseEngine": "disabled",
  "clangd.path": "clangd",
  "clangd.arguments": [
    "--background-index",
    "--clang-tidy",
    "--header-insertion=iwyu",
    "--completion-style=detailed"
  ]
}
```

### 5. Setup Steps

1. Install VSCode extension: `clangd` (llvm-vs-code-extensions.vscode-clangd)
2. Run `direnv allow` in project directory
3. Find your nix store paths using the command above
4. Update `.clangd` with your actual paths
5. Open VSCode: `code .`
6. Restart language server: `Ctrl+Shift+P` → "clangd: Restart language server"

## For Different Projects

When you start a new project:
1. Copy `flake.nix` and `.envrc`
2. Run `direnv allow`
3. Get the paths from `clang++ -E -x c++ - -v < /dev/null 2>&1`
4. Update `.clangd` with the new paths (they change when nix packages update)
5. Copy `.vscode/settings.json`

## Troubleshooting

**"vector file not found", "stdlib.h not found", or similar header errors:**
- Your nix store paths in `.clangd` are wrong/outdated/incomplete
- Run `clang++ -E -x c++ - -v < /dev/null 2>&1 | grep -A 20 "search starts here"` to get fresh paths
- Make sure you have ALL 5 include paths in the EXACT order shown
- Update `.clangd` and restart language server

**clangd not found:**
- Make sure you're in a directory with `.envrc`
- Run `direnv allow`
- Check `which clangd` shows a nix store path

**Changes not taking effect:**
- Always restart the language server after changing `.clangd`
- If that doesn't work, restart VSCode completely

## Why This Works

- `flake.nix`: Provides clang, clangd, and gcc in isolated environment
- `.envrc`: Auto-loads the nix environment when you enter the directory
- `.clangd`: Tells clangd where to find ALL standard library headers in the correct search order
- VSCode settings: Disables conflicting C++ extension and configures clangd

The key insight: clangd needs the EXACT same include paths in the SAME order that `clang++` uses. NixOS doesn't use standard system paths like `/usr/include`, so you must explicitly provide all 5 paths: compiler-rt, C++ headers, platform-specific C++ headers, clang resources, and glibc.