# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

PSAppDeployToolkit is a PowerShell framework for Windows software deployment. It is a dual-language codebase: a PowerShell module (`src/PSAppDeployToolkit`) plus a family of C# projects under `src/PSADT/*` that are compiled and shipped alongside it. `main` is the active integration branch; release branches like `4.1.x` are maintained in parallel.

## Build & Test Commands

The canonical entry point is `Invoke-ADTModuleBuild`, exposed by the `src/PSAppDeployToolkit.Build` helper module.

- Full build: `./build.ps1` (or `build.cmd` on Windows). This runs `Prerequisites → Clean → Dependencies → DotNet → Analyze → UnitTests → Build → IntegrationTests`.
- Subset build: `Import-Module ./src/PSAppDeployToolkit.Build/PSAppDeployToolkit.Build.psd1; Invoke-ADTModuleBuild -Steps Analyze,UnitTests` — the `-Steps` parameter accepts any subset of the phases above.
- Unit tests only (quick loop, no build): `Invoke-Pester ./src/Tests/Unit -Output Detailed` after importing the dev module with `Import-Module ./src/PSAppDeployToolkit/PSAppDeployToolkit.psd1 -Force`.
- Single function's tests: `Invoke-Pester ./src/Tests/Unit/Verb-ADTNoun.Tests.ps1 -Output Detailed`.
- VS Code tasks (`.vscode/tasks.json`) wrap these under labels like `Build`, `BuildNoIntegration`, `Test`, `PesterTest`, `Pester-Single-Detailed`, `Analyze`, `FormattingCheck`, `ValidateRequirements`, `IntegrationTest` — prefer these over ad-hoc commands when validating changes.
- .NET requires a current SDK; the C# projects use `Directory.Build.props` for shared settings (TreatWarningsAsErrors, banned-API analyzer via `src/BannedSymbols.txt`, analysis level `latest-all`).

## Architecture

### Two solutions

- `PSADT.slnx` — all first-party C# projects plus the vendored iNKORE WPF libs in `lib/`.
- `PSADT.Invoke.slnx` — the standalone `Invoke-AppDeployToolkit.exe` launcher under `src/PSADT.Invoke/`.

### PowerShell module layout (`src/PSAppDeployToolkit/`)

- `Public/` — exported functions, one function per file, all using the `ADT` noun prefix.
- `Private/` — internal helpers, scoped `Private:`.
- `PSAppDeployToolkit.psm1` dot-sources every file under `Public/` and `Private/` **only when not compiled**. Release builds inline the function bodies, which is why:
  - One-function-per-file is strict.
  - `$Script:CommandTable` is the preferred way to reference internal commands indirectly (it remains valid after compilation).
- `ImportsFirst.ps1` / `ImportsLast.ps1` bracket the function loading with module-level initialization.
- Most public functions follow the `begin`/`process`/`end` lifecycle with `Initialize-ADTFunction` in `begin` and `Complete-ADTFunction` in `end`; errors flow through `Invoke-ADTFunctionErrorHandler` so `Write-Error -ErrorRecord $_` preserves position info.

### C# projects (`src/PSADT/`)

- `PSADT` — core utilities.
- `PSADT.Interop` — Win32 interop and CsWin32-generated P/Invoke.
- `PSADT.UserInterface` — WPF UI (Fluent + Classic), plus `PSADT.UserInterface.TestHarness` for running the UI standalone.
- `PSADT.ClientServer.{Client,Server}` plus `Client.Launcher` and `*.Compatible` variants — the security-model split that replaces ServiceUI: the SYSTEM-context server never touches the user session, UI work is brokered to a client process running in the user's session.
- `PSAppDeployToolkit` (C# project) — PowerShell-facing C# types referenced from the PS module (attributes like `ValidateNotNullOrWhiteSpace`, etc.).
- `PSADT.Tests` — C# tests.

Most first-party C# projects dual-target .NET Framework 4.7.2 and a current .NET Windows TFM, but check each `.csproj` before assuming.

### Build module (`src/PSAppDeployToolkit.Build/`)

Orchestrates the whole pipeline: `Confirm-ADTScriptEncoding`/`Formatting`/`Integrity`, `Invoke-ADTDotNetCompilation`, `Invoke-ADTPesterUnitTesting`, `Invoke-ADTModuleCompilation`, `Export-ADTScriptTemplate`, `Invoke-ADTPesterIntegrationTesting`, with ADMX template and string-table validation.

### Tests (`src/Tests/`)

- `Unit/` — one `.Tests.ps1` per public function, plus cross-cutting `PSAppDeployToolkit-Module.Tests.ps1` and `ExportedFunctions.Tests.ps1`.
- `Integration/` — machine-touching scenarios.

## Repository-specific Conventions

These come from `.github/instructions/{powershell,pester,csharp}.md` and `.github/copilot-instructions.md`.

### PowerShell

- **PS 5.1 compatibility is hard-required.** No ternary, null-coalescing, pipeline chain, or null-conditional operators. PSScriptAnalyzer enforces the `desktop-5.1.14393.206-windows` compatibility profile.
- Use `[System.Management.Automation.Language.NullString]::Value` when passing null strings to .NET APIs expecting `[string]`.
- UTF-8 **with BOM**, 4-space indent, Allman braces, final newline.
- Use fully-qualified .NET type names in parameters (`[System.String]`, not `[string]`).
- Prefer the module's custom validators: `[PSAppDeployToolkit.Attributes.ValidateNotNullOrWhiteSpace()]`, `[...AllowNullButNotEmptyOrWhiteSpace()]`, `[...ValidateGreaterThanZero()]`.
- Public state-changing commands set `SupportsShouldProcess` and wrap mutations in `$PSCmdlet.ShouldProcess()`.
- Prefer `& { process { } }` pipeline pattern where it already appears; `.Where()`/`.ForEach()` for in-memory collections.
- Public function comment-based help is required; `.NOTES` must state whether an active ADT session is required; declare `[OutputType()]` when returning output.
- Structured errors: `New-ADTErrorRecord` / `New-ADTValidateScriptErrorRecord`; `$PSCmdlet.ThrowTerminatingError()` for fatal cases in `begin`/validation.
- Ignore IDE0028 — it is an IntelliSense bug in this repo context.

### Public contract stability

- On `main`, avoid breaking public parameter names/behaviour/output types; prefer **aliases** over renames; prefer a new function + compat wrapper over silently changing contracts.
- On release branches (`4.1.x`, etc.), avoid breaking public contracts entirely.

### Pester (v5)

- Standard import at the top of every test file:

  ```powershell
  BeforeAll {
      Remove-Module PSAppDeployToolkit -Force -ErrorAction SilentlyContinue
      Import-Module "$PSScriptRoot\..\..\PSAppDeployToolkit\PSAppDeployToolkit.psd1" -Force
  }
  ```

  Do not `Set-Location` to make imports work. Use `BeforeDiscovery` when `-ForEach` data depends on the imported module.
- Prefer `$TestDrive` / `TestRegistry:\` over real filesystem/registry. Unit tests must not write to real HKLM or system dirs.
- Always pass `-ModuleName PSAppDeployToolkit` when mocking PSADT commands. `Write-ADTLogEntry` is commonly mocked.
- Test the public contract, not incidental implementation details. Prefer specific assertions (exception type, stable message fragment, error ID) over bare `Should -Throw`.

## Key Files

- `src/PSAppDeployToolkit/PSAppDeployToolkit.psd1` — module manifest (current version 4.2.0, `PowerShellVersion = 5.1.14393.0`).
- `src/PSAppDeployToolkit/PSAppDeployToolkit.psm1` — module entry point (dev-time dot-sourcing).
- `src/PSAppDeployToolkit.Build/Public/Invoke-ADTModuleBuild.ps1` — the build phase orchestrator.
- `Directory.Build.props` — shared C# settings (TreatWarningsAsErrors, banned-API analyzer, net472 ARM64 pref).
- `src/BannedSymbols.txt` — banned-API list enforced by the analyzer.
- `.vscode/PSScriptAnalyzerSettings.psd1` — analyzer configuration.
- `.github/workflows/module-build.yml` — CI pipeline.
- `.github/instructions/{powershell,pester,csharp}.md` — authoritative language conventions.
