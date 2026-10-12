This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# SOLIDWORKS PDM Professional SDK (PDMSDK)

This is the open-source `BlueByte.SOLIDWORKS.PDMProfessional.SDK` framework — a managed .NET wrapper around the COM-based SOLIDWORKS PDM Professional API (`EPDM.Interop.epdm`). It is shipped to NuGet as `BlueByte.SOLIDWORKS.PDMProfessional.PDMSDK` and is also published as a Visual Studio extension ("PDM SDK") that adds C# and VB.NET add-in project templates. Live docs: <https://bluebytesystemsinc.github.io/SOLIDWORKS-PDM-API-SDK/docs/getting-started.html>.

## Build & restore commands

There is no `dotnet` / MSBuild wrapper script. The solutions are old-style `.sln` files targeting .NET Framework 4.6.1 (SDK) and 4.7.2 (UnitTesting). Build with Visual Studio or `msbuild` from a Developer Command Prompt:

```bat
:: Restore NuGet packages for the SDK (packages.config-based, requires NuGet.exe on PATH or VS)
nuget restore SDK.sln
msbuild SDK.sln /p:Configuration=Debug

:: Build the VS template extension (produces Release\VSExtension.vsix)
nuget restore VSTemplate\SOLIDWORKSPDMAddIn.sln
msbuild VSTemplate\SOLIDWORKSPDMAddIn.sln /p:Configuration=Release
```

Key constraints baked into the projects:
- `SignAssembly=true` for both libraries; SDK uses `PDMFramework.snk`, Services uses `Key.snk`. Do **not** remove signing or strip the key files — PDM loads the add-in via COM and strong-name binding.
- The SDK csproj references `EPDM.Interop.epdm` (PKT `0625a0c4bb5de148`) and `EPDM.Interop.EPDMResultCode` from `BlueByte.SOLIDWORKS.PDMProfessional.Interops.2021.0.0` — that package is the proprietary COM wrapper around `ConisioLib.dll`. Without it the projects will not compile.
- Dependencies are managed via `packages.config` (classic), **not** PackageReference. Do not migrate without also providing the `packages/` folder or switching `msbuild` to use `<PackageReference>`.
- Solutions present: `SDK.sln` (SDK only), `VSTemplate\SOLIDWORKSPDMAddIn.sln` (template + VSIX), `UnitTesting\UnitTesting\UnitTesting.sln`. There is no CI workflow for builds; only `.github/workflows/static.yml` publishes the prebuilt `docs/` site to GitHub Pages.

There is no lint stage (no StyleCop, EditorConfig, or analyzers). There is no automated test project in this open-source repo — `UnitTesting/` references the proprietary `BlueByte.SOLIDWORKS.PDMProfessional.UnitTesting` and `Orca` NuGet packages plus a license-key file at `P:\Blue Byte Systems, Inc\PDMFrameworkUnitTesting\…`, so it only builds inside Blue Byte Systems Inc.

## Repository layout

```
SDK.sln
src/
  BlueByte.SOLIDWORKS.PDMProfessional.SDK/        # core framework (BlueByte.SOLIDWORKS.PDMProfessional.SDK.csproj)
  BlueByte.SOLIDWORKS.PDMProfessional.Services/   # companion library (BlueByte.SOLIDWORKS.PDMProfessional.Services.csproj)
VSTemplate/
  PDM SDK Add-in Template/                        # C# project + .vstemplate (exported zip)
  PDM SDK Add-in Template VBNET/                  # VB.NET counterpart
  SOLIDWORKSPDMAddIn/                             # C# sample add-in (basis for the C# template)
  SOLIDWORKSPDMAddInVBNET/                        # VB.NET sample add-in
  ProjectTemplateWizard/                          # WPF IWizard that collects Name/Description/Company/IsTask
  VSExtension/                                    # VSIX project (source.extension.vsixmanifest, targets VS 2019 + 2022)
  VisualStudioExtension/                          # legacy VSIX (not shipped by the current manifest)
UnitTesting/
  UnitTesting/                                    # MSTest-style console runner against a live vault
  TestAddin/                                      # sample add-in referenced by the test project
docs/                                             # DocFX-generated site (already built — do NOT hand-edit)
docfx/                                            # DocFX source (index.md, toc.yml, docfx.json)
.github/workflows/static.yml                     # deploys docs/ to GitHub Pages on push to master
```

`docs_backup/` is an older DocFX tree that no longer matches the current `docfx/docfx.json` source path — keep for reference, regenerate via `docfx` from `docfx/` if a fresh site is needed.

## SDK architecture (`src/BlueByte.SOLIDWORKS.PDMProfessional.SDK`)

`AddInBase` (`Core/AddInBase.cs`) is the only class an add-in author needs to inherit from. It implements `IEdmAddIn5` (the PDM COM entry point) and orchestrates every concern an add-in has:

1. **Identity / metadata** — reads class-level attributes (`Name`, `Description`, `CompanyName`, `RequiredVersion`, `AddInVersion`, `IsTask`, `Guid`, `ComVisible`) inside `GetAddInInfo`, throws `IdentityInfoException` if any are missing or invalid, and populates a public `Identity` struct (`Core/Identity.cs`).
2. **Hook + command registration** — `[ListenFor(EdmCmdType)]` controls `poCmdMgr.AddHook` calls; `[Menu(...)]` controls `poCmdMgr.AddCmd`. For task add-ins (`IsTask(true)`), `AddInBase` automatically registers the six task hooks (`EdmCmd_TaskSetup/SetupButton/Launch/LaunchButton/Details/Run`). `[CommandVisibility(...)]` gates commands by user/group names or by `EdmSysPerm` rights.
3. **Command dispatch** — `OnCmd` calls the base, which then reflects for methods tagged `[Handles(EdmCmdType, commandID)]` and `Invoke`s them, so add-ins can subscribe to events with ordinary methods.
4. **DI container** — `RegisterTypes()` builds a `SimpleInjector.Container`, auto-registers every exported type whose namespace contains `"Pages."`, and binds the chosen `ILogger` implementation as a singleton. Override `OnRegisterAdditionalTypes(container)` to add your own services; override `OnLoadAdditionalAssemblies(dir)` / `OnUnhandledExceptions` / `OnLoggerTypeChosen` / `OnLoggerOutputSat` for the four other lifecycle hooks documented in the template's `AddIn.cs`.
5. **Cancellation/suspension** — `CheckForCancellationOrSuspension` and `UpdateTaskMessage`/`SetRange` translate to PDM task status state-machine transitions; throw `CancellationException` (`Core/CancellationException.cs`) when the user cancels, `TaskFailedException` (`Core/TaskFailedException.cs`) for unrecoverable failures.
6. **Task pages** — `ITaskPage` + `TaskPage<TViewModel>` (`Core/ITaskPage.cs`, `Core/TaskPage.cs`) handle the setup/details UI; the generic `TaskPage<T>` derives from `UserControl`, owns a `SimpleInjector.Container`, and serializes its view-model through `Newtonsoft.Json` into the task's `GetValEx`/`SetValEx` slot (or into a vault variable if `SaveLoadDataToVariable=true`). `TaskPage.LoadSettings/SaveSettings` persist the JSON to a `.edmtdf` file on disk via `OpenFileDialog`/`SaveFileDialog`.
7. **Logger** — pick one of `ConsoleLogger`, `FileLogger`, `SQLLogger`, `PDMLogger` (`Diagnostics/`); `LoggerType_e` is inferred from the class name (`LoggerBase.GetLoggerType`). `PDMLogger` writes through `IEdmDictionary5` (vault-backed JSON entries via `PDMLoggerHelper`) and is the one used by the sample `TestAddin`. Override `OnPDMLoggerInitialized` to set `LogType = typeof(YourEntryClass)` when you select `LoggerType_e.PDM`.

`AssemblyResolution/` is a vendored subset of Xarial Pty Limited's **xToolkit** (the file headers make this explicit). It supplies the `AppDomain.AssemblyResolve` plumbing (`Helpers/AssemblyResolver` + `LocalFolderReferencesResolver`) so an add-in loaded by PDM can find the SDK's own DLLs alongside its DLL. The `AssemblyMatchFilter_e` enum controls whether matching considers `PublicKeyToken`, `Culture`, and/or `Version`. `AddInBase`'s constructor wires a `LocalFolderReferencesResolver` rooted at the add-in's directory.

`Services/AssemblyResolution/Reflection/` contains the same xToolkit helpers used for binding-redirect lookup (`AppConfigBindingRedirectReferenceResolver`) and reflection helpers (`TypeExtension.TryFindGenericType`, `Lambda.InvokeGenericMethod`, `ResourceHelper.GetResource`, `EnumExtension.TryGetAttribute`).

## Services architecture (`src/BlueByte.SOLIDWORKS.PDMProfessional.Services`)

Independent companion library with three pieces:

- `SettingsManager` + `ISettingsManager` (`SettingsManager.cs`) — vault-scoped settings store. `Initialize(IEdmVault5 vault, string addInName)` then `GetSettings<T>`/`SaveSettings<T>` use `IEdmDictionary5` (via `SettingsManagerHelper`'s extension methods) to read/write JSON per type. `GetUserSettings<T>(int userID)` / `SaveUserSettings<T>(T, int userID)` allow per-user overrides. Requires `using Newtonsoft.Json`.
- `NotifiableBase` — minimal `INotifyPropertyChanged` base class.
- `Win32Window` — `IWin32Window` wrapper for `IEdmVault.MsgBox(int parentHwnd, …)` calls.
- `TypeConverters/` — six `TypeConverter`s (`BOMLayoutNamesConverter`, `StateNamesConverter`, `StringArrayTypeConverter`, `TransitionNamesConverter`, `VariableNamesConverter`, `WorkflowNamesConverter`) plus `CheckedListBoxUITypeEditor`. Each names-converter exposes a `public static string[] Names` (or `ItemsSource`) that the consumer must populate from the vault before the settings property grid is shown — they are not auto-populated by the SDK. The C# project in `VSTemplate/PDM SDK Add-in Template/Task/SetupPages/SettingsViewModel.cs` shows the typical pattern: assign `VariableNamesConverter.VariableNames = ...` from the vault in your add-in startup.

## Add-in author flow

`VSTemplate/PDM SDK Add-in Template/AddIn.cs` is the canonical example of a finished add-in. Required class-level attributes (enforced by `AddInBase.GetAddInInfo`):

```csharp
[Name("…")] [Description("…")] [CompanyName("…")]
[AddInVersion(false, 1)]
[IsTask(false)]
[RequiredVersion(Year_e.PDM2021, ServicePack_e.SP0)]   // or (major:int, minor:int)
[ComVisible(true)] [Guid("…")]
public partial class AddIn : AddInBase { … }
```

`[Menu(commandID, caption, flags)]` declares a PDM command; `[ListenFor(EdmCmdType.EdmCmd_Menu)]` opts in to its hook. Inside `OnCmd`, dispatch on `poCmd.mlCmdID` against a private `Commands` enum and use `ForEachFile(ref ppoData, file => …)` to walk affected files safely — non-file entries are skipped. Build output: enable *Register for COM interop* in project Properties → Build; copy `bin\<Config>\*.dll` into the PDM add-ins folder and register with `regasm /codebase`. The `docs/docs/building_project.html` page (already generated) walks through this.

`VSTemplate/ProjectTemplateWizard/ProjectWizard.cs` collects Name / Description / Company / IsTask via WPF `WizardWindow.xaml` and substitutes them into the `$addinname$` / `$addindescription$` / `$addincompanyname$` / `$addintask$` tokens in `AddIn.cs` and `SOLIDWORKSPDMAddIn.csproj`. The packaged `.zip`s at `VSTemplate/PDM SDK Add-in Template.zip` and `VSTemplate/PDM SDK Add-in Template VBNET.zip` are what `VSExtension/source.extension.vsixmanifest` ships (targets VS 2019 `[14.0,17.0)` and 2022 `[17.0,18.0)`).

## Reference points

- API reference site (do not regenerate by hand): `docs/api/`. Key types: `BlueByte.SOLIDWORKS.PDMProfessional.SDK.AddInBase`, `Core.ITaskPage`, `Core.TaskPage-1`, `Diagnostics.ILogger`, `Diagnostics.IPDMLogger`.
- Conceptual guide: `docs/docs/` (`getting-started.html`, `addinbase_structure.html`, `commands.html`, `logger.html`, `dependencyinjection.html`, `taskadd-ins.html`, plus `TaskSetup.html`, `TaskSetupButton.html`, `TaskLaunch.html`, `TaskRun.html`, `TaskDetails.html`).
- `docfx/docfx.json` is the DocFX source config; the SDK build is rooted at a hard-coded absolute path (`C:\\Users\\jlili\\source\\repos\\…`) — overwrite `src` before re-running DocFX on a different machine.
- NuGet: `BlueByte.SOLIDWORKS.PDMProfessional.PDMSDK` (note: nuget package id differs from the assembly name — install command from README is `Install-Package BlueByte.SOLIDWORKS.PDMProfessional.PDMSDK`).
- `LICENSE.md` is a custom MIT-style license that *withholds* permission from SOLIDWORKS Corp, SOLIDWORKS resellers/partners, and commercializers of SOLIDWORKS-integrating products unless they bought Blue Byte professional services — keep this restriction in any redistribution.

## Common gotchas

- `TestAddin` references `BlueByte.SOLIDWORKS.PDMProfessional.Services.Views.PDMLogView` — that class is **not** in this open-source repo; it ships through a closed-source Blue Byte NuGet. Don't go looking for it on disk.
- `LoggerBase.GetLoggerType()` infers the enum from the class name (`FileLogger` → `LoggerType_e.File`, etc.). Rename any logger subclass to match, or override.
- `MenuAttribute.Flags` is an `int`, not the `EdmMenuFlags` enum — the SDK exposes the higher-bit helpers as `PDMSDKEdmMenuFlags` (`Enums/PDMSDKEdmMenuFlags.cs`); the canonical value `43` in the template is `EdmMenu_OnlyFilesInFileExplorer`.
- `RequiredVersion` enforces `Major >= 6` (`Year_e.PDM2006`+); lower values raise `IdentityInfoException` during `GetAddInInfo`.
- `app.config` binding redirects are not honored for plugin DLLs by PDM's host — the SDK ships `AppConfigBindingRedirectReferenceResolver` precisely to handle that case.