# Debug Browser Build Guide

This fork contains a DEBUG-only development setup for running Safe Exam Browser as a normal, windowed browser during local experiments.

These changes are intended for local development builds only. They are guarded by `#if DEBUG` and should not be used as a production or exam configuration.

## What the Debug Build Changes

- Starts the main browser in a regular resizable window instead of full-screen kiosk mode.
- Keeps the close button available in the debug windowed browser mode.
- Uses permissive default debug settings:
  - no kiosk mode
  - empty application blacklist
  - display validation errors ignored
  - virtual machine policy allowed
  - clipboard allowed
  - browser toolbar, address bar, navigation, and developer console enabled
- Allows Discord and other desktop applications to keep running because the debug defaults clear the blacklist.
- Skips the display-capture guard in debug mode so SEB windows can be visible in screen sharing software.
- Uses capture-friendly Chromium/CefSharp rendering flags in debug mode to make browser content easier to capture.

## Debug Environment Switches

All switches below are only used by DEBUG builds.

| Variable | Default | Effect |
| --- | --- | --- |
| `SEB_DEBUG_WINDOWED_BROWSER` | enabled | Set to `0` to disable the regular windowed browser mode. |
| `SEB_DEBUG_IGNORE_DISPLAY_ERRORS` | enabled | Set to `0` to restore normal display validation failures. |
| `SEB_DEBUG_ALLOW_SCREEN_CAPTURE` | enabled | Set to `0` to restore `WDA_EXCLUDEFROMCAPTURE` window guarding. |
| `SEB_DEBUG_CAPTURE_FRIENDLY_BROWSER` | enabled | Set to `0` to restore default Chromium GPU/compositor rendering. |
| `SEB_DEBUG_ALLOW_DISCORD` | disabled | Allows Discord blacklist entries if a custom config reintroduces them. The default debug config already clears the blacklist. |

Example PowerShell override:

```powershell
$env:SEB_DEBUG_CAPTURE_FRIENDLY_BROWSER = "0"
```

## Build Prerequisites

- Visual Studio 2022 Build Tools with .NET desktop build tools
- .NET Framework 4.8 developer/targeting pack
- NuGet package restore
- WiX Toolset 3.x

The local MSI used during testing was built with portable WiX 3.14 under `.tools/wix314`, but `.tools/` is intentionally ignored by git.

## Restore Packages

```powershell
nuget restore SafeExamBrowser.sln -PackagesDirectory packages
```

If PackageReference projects need restore as well:

```powershell
$msbuild = "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\MSBuild\Current\Bin\MSBuild.exe"
& $msbuild SafeExamBrowser.SystemComponents\SafeExamBrowser.SystemComponents.csproj /t:Restore /p:Configuration=Debug /p:Platform=x64
```

## Build Debug MSI

Set paths first:

```powershell
$msbuild = "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\MSBuild\Current\Bin\MSBuild.exe"
$wix = (Resolve-Path ".tools\wix314").Path
```

Build once so project outputs are fresh:

```powershell
& $msbuild Setup\Setup.wixproj /m /t:Build /p:Configuration=Debug /p:Platform=x64 /p:WixTargetsPath="$wix\wix.targets" /p:WixInstallPath="$wix" /p:WixToolPath="$wix" /p:WixExtDir="$wix" /p:SignOutput=false /p:PreBuildEvent= /p:PostBuildEvent= /v:m
```

Copy the full client output into the runtime application payload. This is required because the setup harvests `SafeExamBrowser.Runtime\bin\x64\Debug`, while the installed client also needs browser and UI dependencies from `SafeExamBrowser.Client\bin\x64\Debug`.

```powershell
Copy-Item -Path "SafeExamBrowser.Client\bin\x64\Debug\*" -Destination "SafeExamBrowser.Runtime\bin\x64\Debug" -Recurse -Force
```

Regenerate WiX component files:

```powershell
& "$wix\heat.exe" dir "SafeExamBrowser.Runtime\bin\x64\Debug" -nologo -ag -g1 -scom -srd -sreg -cg ApplicationComponents -dr ApplicationDirectory -sfrag -var var.SafeExamBrowser.Runtime.TargetDir -out "Setup\Components\Application.wxs" -t "Setup\Components\Application.xslt"
& "$wix\heat.exe" dir "SebWindowsConfig\bin\x64\Debug" -nologo -ag -g1 -scom -srd -sreg -cg ConfigurationComponents -dr ConfigurationDirectory -sfrag -var var.SebWindowsConfig.TargetDir -out "Setup\Components\Configuration.wxs" -t "Setup\Components\Configuration.xslt"
& "$wix\heat.exe" dir "SafeExamBrowser.ResetUtility\bin\x64\Debug" -nologo -ag -g1 -scom -srd -sreg -cg ResetComponents -dr ResetDirectory -sfrag -var var.SafeExamBrowser.ResetUtility.TargetDir -out "Setup\Components\Reset.wxs" -t "Setup\Components\Reset.xslt"
& "$wix\heat.exe" dir "SafeExamBrowser.Service\bin\x64\Debug" -nologo -ag -g1 -scom -srd -sreg -cg ServiceComponents -dr ServiceDirectory -sfrag -var var.SafeExamBrowser.Service.TargetDir -out "Setup\Components\Service.wxs" -t "Setup\Components\Service.xslt"
```

Build the MSI again:

```powershell
& $msbuild Setup\Setup.wixproj /m /t:Build /p:Configuration=Debug /p:Platform=x64 /p:WixTargetsPath="$wix\wix.targets" /p:WixInstallPath="$wix" /p:WixToolPath="$wix" /p:WixExtDir="$wix" /p:SignOutput=false /p:PreBuildEvent= /p:PostBuildEvent= /v:m
```

The unsigned debug MSI is produced at:

```text
Setup\bin\x64\Debug\Setup.msi
```

## Troubleshooting

If the installed browser fails with a missing assembly such as `SafeExamBrowser.Applications.dll`, repeat the full client-to-runtime copy step before harvesting and rebuilding the MSI.

If screen sharing software shows only a splash screen or a blank browser area, make sure the installed build is the latest debug MSI and `SEB_DEBUG_ALLOW_SCREEN_CAPTURE` and `SEB_DEBUG_CAPTURE_FRIENDLY_BROWSER` are not set to `0`.

If display validation fails at startup, make sure the build is DEBUG and `SEB_DEBUG_IGNORE_DISPLAY_ERRORS` is not set to `0`.
