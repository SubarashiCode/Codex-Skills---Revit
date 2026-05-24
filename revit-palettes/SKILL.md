---
name: revit-palettes
description: Build and fix Revit API add-ins that expose modern WPF palettes, dockable panes, floating Family Editor palettes, ribbon tabs/buttons/icons, Revit 2025 manifests, local deployment, Authenticode publisher behavior, or Ahmed Abdalla vendor metadata. Use for tasks involving Revit palettes, Revit dockable UI, Revit Family Editor palette compatibility, palette resize/scrollbar bugs, and Revit add-in signing/deployment.
---

# Revit Palettes

Use this skill when building or repairing Revit add-ins that create palette-style UI. Prefer the repo's existing patterns, but preserve the project lessons below unless the user explicitly asks otherwise.

## Defaults

- Target Revit 2025 with `net8.0-windows`, WPF enabled, and x64 output.
- Reference local Revit assemblies from `C:\Program Files\Autodesk\Revit 2025\RevitAPI.dll` and `RevitAPIUI.dll` with `<Private>false</Private>`.
- Use vendor info for Ahmed Abdalla:
  - `VendorId`: `AHMD`
  - `VendorDescription`: `Ahmed Abdalla`
  - assembly `Authors`/`Company`: `Ahmed Abdalla`
- Revit's security dialog `Publisher` is not controlled by `.addin` vendor fields. It comes from Windows Authenticode signing. Do not claim this is fixed unless the DLL is signed with a trusted certificate.

## Ribbon And Pane Pattern

Create an `IExternalApplication` that:

- Creates/reuses a `Specialist` ribbon tab.
- Creates a panel for the tool, such as `Syntax`.
- Adds a `PushButtonData` pointing to an `IExternalCommand`.
- Registers a dockable pane with `UIControlledApplication.RegisterDockablePane`.
- Uses `DockablePaneProviderData.InitialState.DockPosition = DockPosition.Right` for a native right-docked first load.

Use generated WPF drawing icons when no asset exists. Freeze `DrawingGroup`/`DrawingImage` instances.

## Docking Behavior

Use Revit's native `DockablePane` for any palette that must dock, redock, tab behind Properties/Project Browser, or detect Revit docking zones. A plain WPF `Window` can float on top of Revit, but it will not participate in native Revit docking targets.

Commands should show the native pane first in both project and Family Editor contexts:

```csharp
try
{
    commandData.Application.GetDockablePane(App.PaneId).Show();
    return Result.Succeeded;
}
catch (Exception exception)
{
    message = $"Unable to open the native Revit dockable palette. {exception.Message}";
    return Result.Failed;
}
```

Avoid silent fallback to a floating WPF window unless the user explicitly accepts losing native Revit docking. If a prior version used a floating pane or Revit persisted a bad pane placement, change the `DockablePaneId` GUID to reset the stored Revit pane state and let `DockPosition.Right` apply again.

## WPF Resize Rules

Revit docked panes may be narrower than expected. Avoid layout that pushes the vertical scrollbar beyond the visible pane edge.

- Do not set a large root `MinWidth`; use `MinWidth="0"` or a very small value.
- Set `HorizontalAlignment="Stretch"` and `ClipToBounds="True"` on the palette root.
- Put the vertical `ScrollViewer` at the true right edge. Prefer `Padding="0"` on the `ScrollViewer` and put margins on the inner content.
- Avoid binding an `ItemsControl.Width` to `ScrollViewer.ViewportWidth` when the `ScrollViewer` also has padding.
- Disable horizontal scrolling for the main vertical content: `HorizontalScrollBarVisibility="Disabled"`.
- Add `MinWidth="0"` to star-sized `Grid` columns that contain text.
- Use `TextWrapping="Wrap"` plus `TextTrimming="CharacterEllipsis"` on long text.
- Use a horizontal `ScrollViewer` only for category chips/tool strips.

When a user reports "the slider disappears", they usually mean the vertical scrollbar is clipped because content measured wider than the docked pane. Fix width constraints first.

## Deployment

Build to a fresh deployment folder when Revit has the current DLL locked:

```powershell
dotnet build .\Project.csproj -c Release -o .\Deploy\Revit2025_Ahmed
```

Copy the generated `.addin` to:

```text
%AppData%\Autodesk\Revit\Addins\2025
```

Restart Revit after every DLL change. Revit locks loaded add-in assemblies for the whole session.

## Signing And Publisher

If Revit shows `Unknown Publisher`, inspect:

```powershell
Get-AuthenticodeSignature .\Deploy\...\Addin.dll
```

Fixing this requires signing the DLL with a code-signing certificate trusted by Windows. For local testing, a current-user self-signed code-signing cert can work, but adding it to `Trusted Root Certification Authorities` and `Trusted Publishers` is a persistent trust-store change. Get explicit user approval before creating/trusting such a certificate.

When approved, a local signing workflow can:

1. Create or reuse a current-user code-signing certificate with subject `CN=Ahmed Abdalla`.
2. Import the public certificate into Current User `Root` and `TrustedPublisher`.
3. Sign the deployed DLL with SHA256:

```powershell
Set-AuthenticodeSignature -FilePath .\Deploy\Revit2025_DockableSyntax\SpecialistSyntax.dll -Certificate $cert -HashAlgorithm SHA256
```

4. Verify the exact deployed DLL:

```powershell
Get-AuthenticodeSignature .\Deploy\Revit2025_DockableSyntax\SpecialistSyntax.dll
```

Known local dev certificate from the Specialist Syntax project:

- Subject: `CN=Ahmed Abdalla`
- Thumbprint: `846A0EDCE79E387A9D6A24B05D0B852EF8DA3812`
- Verified status after signing: `Valid`

For public distribution, recommend a commercial code-signing certificate.
