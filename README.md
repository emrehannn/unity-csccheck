# unity-csccheck

Offline, per-assembly C# compile check for Unity projects. **Never launches Unity, never
touches `Temp/UnityLockfile`** — so it works while the editor is open, during Play mode, and
in parallel with other agents writing files.

```
$ unity-csccheck ~/Projects/x
unity-csccheck 1.0.0  ·  /home/emre/Projects/x
  unity   /home/emre/Unity/Hub/Editor/6000.5.3f1/Editor
  scope   assets — 5 of 5 assemblies, 57 sources
  note    42 source(s) found on disk but not in any csproj (Unity has not regenerated them) — included anyway

  OK    Assembly-CSharp            1 files   0.68s
  OK    Assembly-CSharp-Editor     1 files   0.92s
  OK    Game.Editor                2 files   0.98s
  OK    Game.Runtime              46 files   0.92s
  OK    Game.Tests.EditMode        7 files   0.85s

  5 assemblies · 5 ok · 0 failed · 0 warnings · 3.5s
COMPILE OK
```

## Why

An open Unity editor holds `Temp/UnityLockfile`, and *every* `-batchmode` invocation fails
against it — a hard lock, not a race. So there is no way to ask "does my C# compile?" at
exactly the moment you need to. This tool compiles each assembly with Unity's own bundled
Roslyn (`MonoBleedingEdge/lib/mono/4.5/csc.exe`) against Unity's own reference assemblies.

It checks **compilation only**. Asset import, prefab wiring and test runs still need the
editor.

## Install

Requires Python 3.8+ (standard library only) and a Unity install. Runs entirely offline.

```bash
git clone <this repo> ~/Projects/unity-csccheck
ln -s ~/Projects/unity-csccheck/unity-csccheck ~/.local/bin/unity-csccheck
```

## Usage

```bash
unity-csccheck /path/to/UnityProject      # every assembly the project owns
unity-csccheck                             # ... found by walking up from the cwd
unity-csccheck . -a Game.Runtime           # one assembly
unity-csccheck . Assets/Game/Foo.cs        # the assembly that owns this file
unity-csccheck . --list                    # what would be compiled, and from where
unity-csccheck . --scope all               # include Library/PackageCache assemblies
unity-csccheck . -q                        # failures and summary only
UNITY_ROOT=/path/to/Editor unity-csccheck . # override the editor location
```

A single `.cs` file cannot be type-checked on its own, so `--file` mode compiles the whole
assembly that owns it — still under a second for most assemblies.

**Exit codes:** `0` clean · `1` compile errors · `2` tool/setup problem.

## How it works

### The csproj files are the reference model

Unity's generated `*.csproj` record, per assembly, the exact reference set, `DefineConstants`,
`LangVersion` and `AllowUnsafeBlocks` that Unity itself compiles with. The tool replays them:

| csproj element | used as |
|---|---|
| `<Compile Include>` | sources (union'd with on-disk discovery, see below) |
| `<HintPath>` | direct assembly references |
| `<ProjectReference>` | resolved to `Library/ScriptAssemblies/<Name>.dll` |
| `<DefineConstants>` `<LangVersion>` `<AllowUnsafeBlocks>` `<NoWarn>` | compiler flags |

`<ProjectReference>` resolution is not optional. On a DOTS project, Entities/Rukhanka come in
as `<ProjectReference>` to sibling `.csproj` files, **not** as `<HintPath>` — miss them and you
get ~370 phantom errors.

### One flat reference set cannot serve a whole project

This is why the tool is per-assembly rather than one big `csc` call:

- A netstandard2.1 runtime assembly needs `Data/NetStandard/ref/2.1.0/netstandard.dll`, or
  the **default interface members** that `Unity.Entities`' `ISystem` uses fail with **CS8702**.
- A net472 NUnit test assembly needs `UnityReferenceAssemblies/unity-4.8-api/mscorlib.dll`,
  and hits **CS0433** (`HashSet<T> exists in both System.Collections and netstandard`) the
  moment netstandard 2.1 is put in front of it.

Both are true at once in the same project. Unity already solved this per assembly; the csproj
files are that solution written down.

### But the csproj files go stale

They are regenerated on Unity's import, which cannot happen while the editor is busy or
closed — i.e. exactly when this tool is used. In one measured case 42 of a project's 57
sources were invisible to the csproj list. So source membership is **recomputed from disk**
using Unity's own rules:

1. nearest ancestor directory holding an `.asmdef` wins; an `.asmref` joins the assembly it
   points at (GUID form resolved against `Library/PackageCache`);
2. otherwise `Assembly-CSharp`, or `Assembly-CSharp-Editor` when any directory on the path is
   named `Editor`;
3. `Assets/Plugins`, `Assets/Standard Assets`, `Assets/Pro Standard Assets` are the legacy
   first-pass roots;
4. directories that are hidden, end in `~`, or are named `cvs` are ignored.

A brand-new `.asmdef` with no csproj yet borrows `Assembly-CSharp`'s reference set, and says so.

### Dependency order

Assemblies are compiled in dependency waves, and a reference to
`Library/ScriptAssemblies/<X>.dll` is redirected to **this run's freshly built `X.dll`** when X
is also being compiled. Without that, an assembly that depends on one you just edited is
type-checked against Unity's last compile — the stale state being worked around. If a
dependency fails, its dependents fall back to Unity's prebuilt DLL and the report says so.

## Suppressed diagnostics

Known noise is passed to `csc` as `-nowarn:`, which by construction can only suppress
**warnings** — the compiler will not silence an error this way, so nothing here can hide a
real failure. `--no-suppress` turns it off.

| code | why |
|---|---|
| CS0169, CS0414, CS0649 | fields Unity assigns from the inspector; Roslyn sees them as unused |
| **CS0436** | *"type conflicts with the imported type … in Assembly-CSharp"*. Expected and unavoidable: the reference set includes `Library/ScriptAssemblies/*.dll`, which holds Unity's already-compiled copy of the very source being recompiled. Unity never hits this because it compiles a clean graph. Not a defect. |
| CS1701, CS1702 | assembly-version unification chatter |
| CS8632 | nullable annotations in a non-nullable context |

## Limitations — measured, not guessed

- **Roslyn source generators / analyzers are not run.** Harmless for game code: DOTS
  `SystemAPI` / `IJobEntity` / `ISystem` usage in `Assets/` compiles clean without them. It is
  fatal for the *package sources themselves* — at `--scope all`, `Unity.Entities` fails with
  CS0172/CS0173 in `EntityCommandBuffer.cs`, because those files genuinely need
  `Unity.Properties.SourceGenerator`. This is why the default scope is `assets`: package code
  is already compiled in `Library/ScriptAssemblies`, and there is no reason to rebuild it.
- **An unguarded `using UnityEditor;` in a runtime assembly is NOT caught.** Unity's own
  generated csproj for a runtime `.asmdef` carries 67 UnityEditor references, because
  editor-time compilation has to accept `#if UNITY_EDITOR` code. Unity itself only rejects it
  at player-build time. Replaying the csproj faithfully inherits that leniency; catching it
  would need a separate player-profile compile.
- **Requires `Library/ScriptAssemblies/` and the `*.csproj` files to exist** — the project must
  have been opened in Unity at least once. The tool fails with exit 2 and a clear message
  otherwise.
- Compiles for the editor profile only (the defines Unity recorded); it does not check other
  build targets.

## Verified against

| project | assemblies | sources | wall |
|---|---|---|---|
| `synesthesia` (HDRP, 3 asmdefs + Assembly-CSharp) | 5 | 57 | 3.6 s |
| `walkin` (DOTS: Entities 6.5, Rukhanka, CFXR) | 8 | 291 | 4.0 s |

Both `COMPILE OK`, exit 0. Negative test in each — a real syntax error in a scratch file is
reported with file/line and exits 1.
