# DLL Packer — Documentation

**Publisher:** NeikPack  **Version:** 1.0.0

DLL Packer compiles a folder of scripts into dlls, using the C# compiler that ships inside the Unity Editor. Keep
editing your code as normal scripts — breakpoints, Hot Reload and quick iteration all work — and when a version is
ready, pack it to hand to your team, your clients or other projects.

Pick a folder and DLL Packer works out the rest the way Unity does: editor scripts (in an `Editor` folder or an
editor-only asmdef) go into an Editor-only dll, game code goes into a dll that runs in the Editor and on every player.

| Feature | What it does |
|---|---|
| Built-in compiler | Uses the Roslyn compiler of the running Editor. No .NET SDK, no IDE, no package to install. |
| Game and editor code sorted for you | `MyTool.dll` for game code, `Editor/MyTool.Editor.dll` for editor code, like the packages you know. |
| One dll per platform | Code with `#if UNITY_ANDROID`, `#if UNITY_IOS`, `#if UNITY_EDITOR` gets a dll per platform you tick. |
| Include dependencies | Like exporting a package with dependencies: project scripts the code uses come along, packages and plugins become references. |
| Checked against every player | Game code is compiled again against each installed player's own Unity assemblies, so editor-only Unity API is caught before a player build. |
| Checks | Reads the scripts before building and explains what would break the build or the dll, with one-click fixes. |
| `.meta` with platforms and a fixed GUID | Each dll imports for the right platforms anywhere, and keeps the same GUID across versions. |
| Compile against any installed Unity | References and version defines (`UNITY_2022_3_OR_NEWER`…) come from the Unity you pick. |

---

## 1. Requirements

- Unity 2022.3 LTS or newer, including Unity 6.
- No package dependencies. DLL Packer is editor-only: nothing from it is included in player builds.
- Building runs the Editor's own `dotnet` + `csc` as a separate process. Nothing is downloaded and nothing leaves
  your machine.
- To check game code against a player (Android, iOS, Windows, WebGL), that player's build support must be installed
  in the Unity you compile against.

## 2. Folder structure

```
NeikPack/DLLPacker/
├── Documentation/
│   └── DLLPacker_Documentation.md       this file
└── Scripts/
    └── Editor/
        ├── NeikPack.DllPacker.Editor.asmdef   editor-only assembly
        ├── DllPackerWindow.cs                 the window
        ├── PackSession.cs                     settings, scripts, plan, checks and build of one folder
        ├── DllCompiler.cs                     compiler, references, defines, platforms, .meta
        ├── PackBuilder.cs                     runs a build: every dll, the player checks, the .meta files
        ├── DependencyResolver.cs              Include dependencies
        ├── ScriptScanner.cs                   reads scripts: usings, types, #if branches, names
        ├── AssemblyCatalog.cs                 the project's assemblies and the namespaces they declare
        └── PreflightChecks.cs                 the checks
```

The `DLLPacker` folder can be moved anywhere under `Assets`. All code is in the `NeikPack.DllPacker.Editor`
namespace.

## 3. Quick start

1. Open **Tools > DLL Packer**.
2. Select the folder in the Project window and press **Use selection** (or type or browse to it). The name of the
   dll defaults to the folder name.
3. **Output** lists the dlls the build will make and where each one loads. Adjust if needed:
   - **One dll per platform** — shown when game code tests platform symbols.
   - **Include dependencies** — when the code uses project scripts outside the folder.
4. Read **Checks**. Errors there are what the build would trip on; most come with a button (**Add** a reference,
   **Tick** a script, **Include dependencies**).
5. Press **Build**. The output folder (`<project>/Build~/DLLPacker/<Name>/` by default) now holds the dlls, their
   `.meta` files and the non-script files of the folder (README, images…).
6. Copy that folder into the `Assets` folder of another project.

## 4. The window

The window follows the job from top to bottom. Until a folder is picked only **Source** is shown.

### Source

- **Folder** — the folder whose `.cs` files (subfolders included) are packed. It can be inside or outside the project.
- **Name** — the file name and assembly name of the dll. Remembered per folder.

### Output

The dlls the next build makes — file name, folder, the platforms it loads on (tags), and how many scripts it holds.
Hover a row for its full path. Below the list:

- **One dll per platform** — only shown when the game code tests platform symbols; see section 6.
- **Include dependencies** — see section 7. The number of scripts and references it adds is shown next to it.

### Scripts

Every script of the folder with a box to leave it out, and a **Game** / **Editor** tag that says which dll it goes into.
Click a tag to move a script to the other dll (an outlined tag was set by hand; hover it for the reason it got its
role). Double-click a name to open the script. With Include dependencies on, **Dependencies** lists the project scripts
that come along and why.

### References

Unity's own assemblies are always referenced. Add anything else the code uses:

- **Add** opens a searchable list of the project's assemblies — every asmdef assembly and every precompiled plugin —
  grouped by package.
- The folder button references any `.dll` on disk.
- Rows tagged **auto** were added by Include dependencies.
- A green dot means the reference was found; a red dot blocks the build until it is fixed or removed.

### Checks

One line per finding — error, warning or note — with a button when there is a one-click fix. Click a line to read the
explanation and the `File.cs:12` links to the script. Clicking the status line at the bottom of the window scrolls here.

### Advanced

Collapsed by default:

- **Output** — the dlls go to `<folder>/<name>/`. A folder ending with `~` (the default `Build~`) is ignored by Unity.
- **Unity** — the Unity to compile against, with the compiler and class library it gives.
- **Extra defines** — scripting define symbols to add, separated by `;`. **Defines** shows the final list.
- **Write .meta** (on), **Copy README, images and other non-script files** (on), **Check game code against every
  installed player** (on).

### Build bar

**Build** compiles every dll; it is disabled while something blocks the build, and the status line says what.
**Open output** shows the result in the file browser. After a failed build, **Compiler output** lists every error, one
section per dll and per player check; click one to open the script at that line. A build is all or nothing: if one
compile fails, no dll of that build is left in the output.

Settings are stored in EditorPrefs per project — output folder, extra defines, selected Unity, the three options — and
per folder — name, unticked scripts, roles set by hand, references, platforms, Include dependencies. Nothing is written
into the project.

## 5. Game code and editor code

Each script gets the role Unity gives it:

- **Inside the project** — the assembly Unity compiles it into decides: `Assembly-CSharp-Editor` or an asmdef limited
  to the Editor means editor code, anything else game code.
- **Outside the project** — Unity's folder rule: a script under a folder named `Editor` is editor code.

The roles decide the dlls:

| Folder holds | Dlls |
|---|---|
| Only editor scripts (an editor tool) | `<Name>.dll` — Editor only |
| Game code | `<Name>.dll` — Editor and every player (or one per platform, section 6) |
| Game code and editor scripts | `<Name>.dll` as above, plus `Editor/<Name>.Editor.dll` — Editor only, referencing `<Name>.dll` |

Editor code is compiled like Unity compiles editor scripts: `UNITY_EDITOR`, `UnityEditor`, .NET Framework 4.8 API.
Game code is compiled like Unity compiles scripts for a player: **.NET Standard 2.1** (it loads under either API
Compatibility Level) and **UnityEngine only**. In a dll for all platforms, `UNITY_EDITOR` and the platform symbols are
false, so `#else` branches are compiled — code that only uses `UNITY_EDITOR` for editor conveniences usually works as
is. With **Check game code against every installed player** on, the game dll is compiled once more against each
installed player's Unity assemblies: the Editor's `UnityEngine` has API that players do not (for example
`Texture2D.alphaIsTransparency`), and the check moves that failure from the player build to packing time.

## 6. One dll per platform

When game code tests platform symbols, **One dll per platform** appears under Output, its boxes ticked from the symbols
found:

```
<Name>/
├── EditorOnly/<Name>.dll   UNITY_EDITOR, compiled with UnityEditor        .meta: Editor only
├── Android/<Name>.dll      UNITY_ANDROID, Android player assemblies       .meta: Android only
├── iOS/<Name>.dll          UNITY_IOS, iOS player assemblies               .meta: iOS only
└── Other/<Name>.dll        no platform symbol                             .meta: every platform not ticked
```

All the dlls share one assembly name, so scripts and other assemblies reference it as usual; each `.meta` enables a
different set of platforms, so exactly one is loaded in the Editor and in each player build. **Other** is always built,
so switching to a platform that was not ticked (macOS, Linux, consoles…) never leaves the project without the dll. The
Editor variant's folder is `EditorOnly`, not `Editor`: a dll inside a folder named `Editor` would be hidden from game
scripts. Components and `ScriptableObject` classes work across the variants: a scene saved in the Editor points at the
Editor dll, and a player build resolves the same class in its own platform's dll.

| Platform | Defines | Unity assemblies |
|---|---|---|
| Editor | `UNITY_EDITOR`, `UNITY_EDITOR_64` | the Editor's, `UnityEditor` included |
| Android | `UNITY_ANDROID`, `UNITY_ANDROID_API`, `PLATFORM_ANDROID` | Android player |
| iOS | `UNITY_IOS`, `UNITY_IPHONE`, `UNITY_IPHONE_API`, `PLATFORM_IOS`, `PLATFORM_IPHONE`, `ENABLE_IL2CPP` | iOS player |
| Windows | `UNITY_STANDALONE`, `UNITY_STANDALONE_WIN`, `PLATFORM_STANDALONE`, `PLATFORM_STANDALONE_WIN`, `UNITY_64`, `PLATFORM_ARCH_64` | Windows player |
| WebGL | `UNITY_WEBGL`, `PLATFORM_WEBGL`, `ENABLE_IL2CPP` | WebGL player |
| Other | none | the Editor's runtime assemblies, checked against each installed player not ticked |

A platform whose build support is not installed is compiled against the Editor's runtime assemblies instead (its box
is dimmed).

## 7. Include dependencies

A dll cannot reference the project's own scripts (`Assembly-CSharp` and its editor and `-firstpass` siblings): Unity
compiles them after plugins and references every plugin from them. **Include dependencies** compiles the scripts the
code needs into the dll instead, the way exporting a package with dependencies takes the files along:

- **Project scripts** the code uses — a type it names in a namespace it can see, an extension method it calls, a
  dotted name it spells out, a namespace it imports — are added, then the scripts *they* use, until nothing new turns
  up. Game code only takes game scripts; editor code takes both. Scripts you unticked in the folder are never added.
- **Packages and plugins** the code uses become references (tagged **auto**): a dll can reference those.
- If a build still finds a name missing (an extension method, a part of a partial class, an assembly a type forwards
  to), DLL Packer adds what supplies it and builds again — up to three times.

The project's scripts are read once (a few seconds for a thousand scripts) and cached; later searches take a fraction
of a second. The search reads names, not meaning, so it can take a few scripts more than strictly needed, never fewer
than it can see.

Everything included is compiled into the dll. A project that also contains those scripts would have the same classes
twice: ship a dll built with dependencies to projects that do not have them. A note in Checks lists where the included
scripts come from.

## 8. How the dll is compiled

The compiler is the one the Editor uses itself, taken from the running Editor:

| Unity | Compiler |
|---|---|
| 2022.3 – 6000.4 | `Editor/Data/NetCoreRuntime/dotnet` + `Editor/Data/DotNetSdkRoslyn/csc.dll` |
| 6000.5 and newer | `Editor/Data/DotNetSdk/dotnet` + `DotNetSdk/sdk/<version>/Roslyn/bincore/csc.dll` |

The Unity picked under **Advanced > Unity** decides the references (section 5 and 6, plus **References**) and the
version defines, exactly as Unity generates them: `UNITY_2022_3_45`, `UNITY_2022_3`, `UNITY_2022`, every
`UNITY_X_Y_OR_NEWER` up to that version. Machine and project symbols (`UNITY_EDITOR_WIN`, Player Settings symbols)
are left out on purpose; add the ones the code needs under **Extra defines**.

Build against the **oldest** Unity you want to support: a dll built against 2022.3 takes its
`#if !UNITY_6000_0_OR_NEWER` branches and loads on Unity 6 as well, while one built against Unity 6 may call APIs
that 2022.3 does not have.

Every build runs with `-optimize+ -debug- -deterministic` and writes no `.pdb`: no line numbers, local variable names
or source paths end up in the dll, and the same sources built by the same Editor always give the same bytes.

## 9. The .meta file and the GUID

With **Write .meta** on (the default), every build writes `<Name>.dll.meta` next to each dll, in the format Unity
writes for plugins (the same layout as Unity's own `Newtonsoft.Json` editor and player dlls):

- The platforms are set for you: Editor only for editor code, Any Platform for a game dll for all platforms, and the
  split of section 6 otherwise. No `Editor` or `Plugins` folder is needed.
- The GUID is derived from the dll's name (and platform), so every build, every machine and every copy share it. Unity
  refers to a class inside a dll by *dll GUID + class name*, so scenes and assets that use the classes keep working
  when the dll is replaced by a newer build. Always copy the `.meta` together with the dll.

The platform settings are written again at every build. An existing `.meta` keeps its GUID even when it differs from
the fixed one — a project may already reference the dll by it — and the build logs both GUIDs. Delete that `.meta`
only if no project references the dll by the old GUID.

## 10. Checks

The checks read the ticked scripts the way the compiler will — comments skipped, `#if` branches decided with the same
defines as the build — once for the game code and once for the editor code, and run again whenever the scripts, the
roles, the platforms, the defines or the references change.

| Level | Check | Fix |
|---|---|---|
| Error | A `using` whose namespace lives in a package or plugin assembly the build does not reference | **Add** |
| Error | Code from the project's own scripts (`Assembly-CSharp`…) | **Include dependencies**, or move that code into an asmdef |
| Error | A namespace or type declared only in an unticked script | **Tick** |
| Error | Game code using a class from an editor script | Move the class out of the Editor folder, or mark the script Editor |
| Error | Game code using `UnityEditor` outside `#if UNITY_EDITOR` | Wrap it, or mark the script Editor |
| Error | Game code referencing an editor-only assembly | Remove it from References |
| Warning | A namespace no loaded assembly declares | — |
| Warning | A dll for all platforms with `UNITY_ANDROID`, `UNITY_IOS`… branches that are never compiled | Turn on One dll per platform |
| Warning | One dll per platform with platform symbols that have no dll of their own (they stay false in Other) | Tick their platform if offered |
| Warning | Classes deriving from `MonoBehaviour` or `ScriptableObject` | Keep **Write .meta** on (section 9) |
| Warning | `#if` symbols the build does not define | Add them to Extra defines if the dll needs them |
| Warning | String paths into the source folder (`"Assets/MyTool/Icons/…"`) | Find files relative to the dll, or by GUID |
| Note | Scripts compiled in by Include dependencies, by folder | Ship to projects that do not have them |
| Note | `UNITY_EDITOR` code left out of a game dll for all platforms | Tick Editor under One dll per platform if needed |
| Note | Platform branches built per platform | — |
| Note | Version-specific code (`#if UNITY_X_Y_OR_NEWER`) and the value it gets for the chosen Unity | Build once per Unity version if both branches are needed |
| Note | String paths to other assets of the project | — |

After a failed build, errors `CS0246`, `CS0234`, `CS0103`, `CS0012` and `CS1069` are traced to the same causes, so
"type or namespace not found" comes with the reference to add or the script to tick.

## 11. Limitations

- **Do not build into the `Assets` folder of the project that holds the sources.** The dll would define the same
  classes as the scripts and Unity reports duplicate types. Advanced warns when the output is inside `Assets`.
- **References come from the open project**, built by the running Editor, whatever Unity is picked. Every project
  that uses the dll needs the same assemblies.
- **Compile-time decisions are fixed in the dll.** Version branches carry the Unity it was built against; machine
  symbols (`UNITY_EDITOR_WIN`), the scripting backend of Android and Windows (`ENABLE_IL2CPP`, `ENABLE_MONO`),
  `DEVELOPMENT_BUILD` and Player Settings symbols are not defined unless added under Extra defines — code behind them
  is left out, including code of included dependencies. Prefer runtime checks (`Application.platform`,
  `Debug.isDebugBuild`) where they will do.
- **macOS, Linux and other platforms** have no dll of their own: they use the Other dll.
- **IL2CPP code stripping** (iOS, WebGL, usually Android) removes code only reached through reflection, as for any
  script. Add a `link.xml` or `[Preserve]` where the dll uses reflection.
- **Code generated at compile time does not run on a packed dll** unless the build runs it: Roslyn source generators
  (MessagePack, MemoryPack…) and IL post-processing (Netcode for GameObjects, Entities) apply to the scripts Unity
  compiles, not to a precompiled dll.
- **Classes saved in assets.** Assets made while the class was a script point at that script's GUID; once the class
  lives in a dll they must be re-pointed at the dll (or re-created).
- **API differences between Unity versions** are not detected by the checks. Building against that Unity reports them.
- **Compiling against another Unity** needs that Unity installed in the same folder as the running one (the Unity Hub
  layout).
- **A dll is not protection.** .NET assemblies decompile easily (ILSpy, dnSpy, dotPeek). Run an obfuscator over the
  output if you need more, and test the code afterwards: obfuscation can break reflection-based Unity features.

## 12. Troubleshooting

| Message / symptom | Fix |
|---|---|
| `No Roslyn compiler found.` | Use Unity 2022.3 or newer. The compiler ships in `Editor/Data/NetCoreRuntime` + `DotNetSdkRoslyn`, or `Editor/Data/DotNetSdk` from Unity 6000.5. Reinstall the Editor if those folders are missing. |
| `Unity <version> is incomplete; pick another install under Advanced.` | That install lacks its class library or Unity assemblies. Pick another one under **Advanced > Unity**. |
| `The dll compiles, but not against the <platform> player` | The code uses Unity API the Editor has and that player does not. Wrap it in `#if UNITY_EDITOR`, or use **One dll per platform**. |
| `Tick at least one platform, or turn off One dll per platform.` | One dll per platform is on with no platform ticked. |
| `Reference <name> is not in this project; remove it or fix the path.` | The package or dll was removed or renamed. Remove the reference, or add it again from **Add**. |
| `The type or namespace name '…' could not be found` | Look at **Checks**: it names the reference to add, the script to tick, or offers Include dependencies. |
| `The type or namespace name 'UnityEditor' could not be found` in game code | Editor API in game code: wrap it in `#if UNITY_EDITOR`, or click the script's Game tag to make it Editor. |
| Include dependencies takes a large part of the project | The code depends on scripts that depend on much more. Move the shared code into an asmdef and reference it instead, or untick the scripts that pull it in. |
| `Cannot replace <dll> (is it open somewhere?)` / `Cannot prepare the output folder` | Another program — often another Unity that has the dll loaded — locks the file. Close it, or build to another folder. |
| `The compiler did not finish within 300 s and was stopped.` | Something blocked the compiler process (antivirus, a very large folder). Build again; exclude the output folder from real-time scanning if it repeats. |
| Duplicate type errors after building | The output folder is inside `Assets` of the project that holds the sources, or a project has scripts a dll with dependencies also contains. |
| `Multiple precompiled assemblies with the same name` after copying | Two dlls of the same name are enabled for one platform — usually an old copy next to the new folder. Delete the old one; keep the `.meta` files DLL Packer wrote. |
| `Kept the GUID … of the existing <name>.dll.meta` | A `.meta` with another GUID was already in the output folder; see section 9. |

## 13. Version history

- **1.0.0** — First release.

## 14. Support

For questions and bug reports, use the contact details on the NeikPack publisher page on the
Unity Asset Store.
