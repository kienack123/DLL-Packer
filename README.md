# DLL Packer

Pack a folder of scripts into dlls with the C# compiler that ships inside the Unity Editor. Editor and
game code are split the way Unity does it, game code can get one dll per platform, the scripts the code
depends on can come along, and checks explain what would break before you build.

Namespace: `NeikPack.DllPacker.Editor` — Editor only.

## Installation

Add this line to `Packages/manifest.json` under `dependencies`:

```json
"com.kienack123.dll-packer": "https://github.com/kienack123/DLL-Packer.git#1.0.0"
```

or **Window ▸ Package Manager ▸ + ▸ Install package from git URL…** and paste
`https://github.com/kienack123/DLL-Packer.git#1.0.0`.

Drop the `#1.0.0` suffix to always track the latest commit on `main`.

The package ships a precompiled editor-only `DLLPacker.dll` that references nothing but
`UnityEngine` / `UnityEditor`.

## Open it

`Tools ▸ DLL Packer`

## Documentation

The window, the dlls it writes, platforms, dependencies, and every check with its fix are described in
[Documentation~/DLLPacker_Documentation.md](Documentation~/DLLPacker_Documentation.md).
