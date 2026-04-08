---
title: "Building a Resilient build.ps1 for psake Projects"
date: 2026-03-30
draft: false
slug: "resilient-build-ps1"
summary: "A walkthrough of the patterns that make a psake build.ps1 entry point robust — from bootstrap flags and tab completion to dependency locking and proper CI exit codes."
tags: ["PowerShell", "psake", "Automation", "DevOps", "CI/CD"]
categories: ["PowerShell", "DevOps"]
params:
  author1: "Trent Blackburn"
  featured_image: ""
  ai_note: "AI was used to help structure and format this blog post"
---

If you've worked with [psake](https://psake.dev), you know that `psakefile.ps1`
defines your build tasks — but it's `build.ps1` that ties everything together.
It's the single entry point that bootstraps dependencies, invokes psake, and
makes sure CI pipelines get a proper exit code.

The PowerShell community has converged on a solid `build.ps1` pattern. Projects
like [PowerShellBuild](https://github.com/psake/PowerShellBuild),
[PoshBot](https://github.com/poshbotio/PoshBot), and
[psake itself](https://github.com/psake/psake) all share the same core
structure. This post walks through the patterns I've added on top of that
baseline to handle the rough edges you hit in enterprise environments — module
lock contention, missing TLS protocols, internal package feeds, and developer
UX.

## The Community Baseline

Most psake projects use a `build.ps1` that looks something like this:

```powershell
[CmdletBinding(DefaultParameterSetName = 'task')]
param(
    [parameter(ParameterSetName = 'task', Position = 0)]
    [string[]]$Task = 'default',
    [switch]$Bootstrap,
    [parameter(ParameterSetName = 'Help')]
    [switch]$Help
)

$ErrorActionPreference = 'Stop'

if ($Bootstrap.IsPresent) {
    Get-PackageProvider -Name Nuget -ForceBootstrap | Out-Null
    Set-PSRepository -Name PSGallery -InstallationPolicy Trusted
    if (-not (Get-Module -Name PSDepend -ListAvailable)) {
        Install-Module -Name PSDepend -Repository PSGallery -Scope CurrentUser -Force
    }
    Import-Module -Name PSDepend -Verbose:$false
    Invoke-PSDepend -Path './requirements.psd1' -Install -Import -Force -WarningAction SilentlyContinue
}

$psakeFile = './psakeFile.ps1'
if ($PSCmdlet.ParameterSetName -eq 'Help') {
    Get-PSakeScriptTasks -buildFile $psakeFile |
        Format-Table -Property Name, Description, Alias, DependsOn
} else {
    Set-BuildEnvironment -Force
    Invoke-psake -buildFile $psakeFile -taskList $Task -nologo
    exit ([int](-not $psake.build_success))
}
```

This gives you a lot out of the box: **`-Bootstrap`** installs
[PSDepend](https://github.com/RamblingCookieMonster/PSDepend) and your declared
dependencies, **`-Help`** lists available tasks,
**`Set-BuildEnvironment`** makes the script CI-aware, and
**`exit ([int](-not $psake.build_success))`** translates psake's success boolean
into a proper exit code so your pipeline fails when the build fails.

Credit where it's due — this is a solid foundation. You can find real-world
examples in [PowerShellBuild](https://github.com/psake/PowerShellBuild),
[PoshBot](https://github.com/poshbotio/PoshBot), and
[devblackops/github-action-psscriptanalyzer](https://github.com/devblackops/github-action-psscriptanalyzer).
The rest of this post covers what I've added on top.

## Clear Errors When Bootstrap Is Skipped

In the standard pattern, if a developer skips `-Bootstrap` and PSDepend isn't
installed, the script fails with an opaque error — typically something like
`Invoke-PSDepend: The term 'Invoke-PSDepend' is not recognized`. That's not
helpful for someone who just cloned the repo.

The fix is a simple guard:

```powershell
if ($Bootstrap) {
    # ... install dependencies ...
}
else {
    if (-not (Get-Module -Name 'PSDepend' -ListAvailable)) {
        throw 'Missing dependencies. Please run with the "-Bootstrap" flag to install dependencies.'
    }
    Invoke-PSDepend -Path $PSScriptRoot -Recurse $False -WarningAction 'SilentlyContinue' -Import -Force
}
```

Now the developer gets a one-line message telling them exactly what to do. Small
change, big improvement in onboarding experience.

## Dynamic Tab Completion

The standard `build.ps1` handles task names in one of two ways:
**`[ValidateSet()]`** with a hardcoded list (used by PoshBot, psake itself), or
**`[ArgumentCompleter]`** that calls `Get-PSakeScriptTasks` (used by
PowerShellBuild).

`[ValidateSet()]` works, but you have to update it every time you add or rename a
task. `[ArgumentCompleter]` reads the task list live from your psake file:

```powershell
[ArgumentCompleter( {
        param($Command, $Parameter, $WordToComplete, $CommandAst, $FakeBoundParams)
        try {
            Get-PSakeScriptTasks -BuildFile './build.psake.ps1' -ErrorAction 'Stop' |
            Where-Object { $_.Name -like "$WordToComplete*" } |
            Select-Object -ExpandProperty 'Name'
        }
        catch {
            @()
        }
    })]
[string[]]$Task = 'default',
```

The `try/catch` returning `@()` is important — it means tab completion degrades
gracefully if psake isn't installed yet (before the first `-Bootstrap` run)
instead of throwing an error in the user's terminal.

One gotcha: PSScriptAnalyzer will flag the completer's parameters (`$Command`,
`$Parameter`, `$CommandAst`, `$FakeBoundParams`) as unused, even though they're
required by the `[ArgumentCompleter]` contract. You'll need
`SuppressMessageAttribute` declarations at the top of the script:

```powershell
[Diagnostics.CodeAnalysis.SuppressMessageAttribute(
    'PSReviewUnusedParameter',
    'Command',
    Justification = 'false positive'
)]
```

Repeat for each parameter. It's verbose, but it keeps your PSScriptAnalyzer
output clean.

## Try-Import-First Pattern

This is the pattern I haven't seen in any other `build.ps1` — and it's the one
that's saved me the most headaches.

The standard bootstrap calls `Invoke-PSDepend -Install -Import`, which downloads
and imports modules in one shot. That works fine for a single developer, but in
CI with concurrent jobs sharing a module cache, you can hit file lock errors when
one job is mid-install while another tries to do the same.

The fix: **try importing first, only install if the import fails**.

```powershell
$psDependParameters = @{
    Path          = $PSScriptRoot
    Recurse       = $False
    WarningAction = 'SilentlyContinue'
    Import        = $True
    Force         = $True
    ErrorAction   = 'Stop'
}

$importSucceeded = $false
try {
    Invoke-PSDepend @psDependParameters
    $importSucceeded = $true
    Write-Verbose 'Successfully imported existing modules.' -Verbose
}
catch {
    Write-Verbose "Could not import all required modules: $_" -Verbose
    Write-Verbose 'Attempting to install missing or outdated dependencies...' -Verbose
}

if (-not $importSucceeded) {
    try {
        Invoke-PSDepend @psDependParameters -Install
    }
    catch {
        Write-Error "Failed to install and import required dependencies: $_"
        Write-Error 'This may be due to locked module files. Please restart the build environment or clear module locks.'
        if ($_.Exception.InnerException) {
            Write-Error "Inner exception: $($_.Exception.InnerException.Message)"
        }
        throw
    }
}
```

If the modules are already present from a previous run (or a parallel job that
finished first), the import-only path is instant and lock-free. You only pay the
install cost when something is actually missing or outdated. The error handling
also gives CI operators a clear next step when lock contention does occur.

## Internal Repository Registration

Enterprise teams often host internal NuGet feeds — ProGet, Azure Artifacts,
MyGet, or similar — rather than pulling everything from PSGallery. The bootstrap
needs to register that repository before PSDepend can install from it.

The pattern is idempotent: check if the repo exists, register it if it doesn't.

```powershell
$repositoryName = 'internal-nuget-repo'
if (-not (Get-PSRepository -Name $repositoryName -ErrorAction 'SilentlyContinue')) {
    $repositoryUrl = "https://nuget.example.com/api/v2/$repositoryName"
    $registerPSRepositorySplat = @{
        Name                      = $repositoryName
        SourceLocation            = $repositoryUrl
        PublishLocation           = $repositoryUrl
        ScriptSourceLocation      = $repositoryUrl
        InstallationPolicy        = 'Trusted'
        PackageManagementProvider = 'NuGet'
    }
    Register-PSRepository @registerPSRepositorySplat
}
```

One detail worth calling out: before registering, you may need to patch the TLS
protocol set. Some older Windows versions default to TLS 1.0/1.1, which modern
NuGet feeds reject. The key is to use `-bor` (bitwise OR) to *add* TLS 1.2 and
1.3 without removing whatever protocols are already enabled:

```powershell
[System.Net.ServicePointManager]::SecurityProtocol = (
    [System.Net.ServicePointManager]::SecurityProtocol -bor
    [System.Net.SecurityProtocolType]::Tls12 -bor
    [System.Net.SecurityProtocolType]::Tls13
)
```

Using `-bor` instead of assignment (`=`) means you don't break connections that
legitimately need an older protocol. It's a one-liner that prevents a class of
mysterious "unable to connect" errors in mixed-OS environments.

## PowerShellGet Version Pinning

If your bootstrap registers internal repositories, you need PowerShellGet v2.
Version 3 changed the module registration API and may not be available in all
environments. Pinning to v2 avoids surprises:

```powershell
$powerShellGetModule = Get-Module -Name 'PowerShellGet' -ListAvailable |
    Where-Object { $_.Version.Major -eq 2 } |
    Sort-Object -Property 'Version' -Descending |
    Select-Object -First 1

$powerShellGetModuleParameters = @{
    Name           = 'PowerShellGet'
    MinimumVersion = '2.0.0'
    MaximumVersion = '2.99.99'
    Force          = $true
}

if (-not $powerShellGetModule) {
    Install-Module @powerShellGetModuleParameters -Scope 'CurrentUser' -AllowClobber
}
Import-Module @powerShellGetModuleParameters
```

The `MinimumVersion`/`MaximumVersion` range ensures you get the latest v2.x
without accidentally pulling in v3. `AllowClobber` handles the case where a
different version is already loaded.

## The Complete Script

Here's everything above combined into a single `build.ps1`:

```powershell
[Diagnostics.CodeAnalysis.SuppressMessageAttribute(
    'PSReviewUnusedParameter',
    'Command',
    Justification = 'false positive'
)]
[Diagnostics.CodeAnalysis.SuppressMessageAttribute(
    'PSReviewUnusedParameter',
    'Parameter',
    Justification = 'false positive'
)]
[Diagnostics.CodeAnalysis.SuppressMessageAttribute(
    'PSReviewUnusedParameter',
    'CommandAst',
    Justification = 'false positive'
)]
[Diagnostics.CodeAnalysis.SuppressMessageAttribute(
    'PSReviewUnusedParameter',
    'FakeBoundParams',
    Justification = 'false positive'
)]
[CmdletBinding(DefaultParameterSetName = 'task')]
param(
    [parameter(ParameterSetName = 'task', Position = 0)]
    [ArgumentCompleter( {
            param($Command, $Parameter, $WordToComplete, $CommandAst, $FakeBoundParams)
            try {
                Get-PSakeScriptTasks -BuildFile './psakeFile.ps1' -ErrorAction 'Stop' |
                Where-Object { $_.Name -like "$WordToComplete*" } |
                Select-Object -ExpandProperty 'Name'
            }
            catch {
                @()
            }
        })]
    [string[]]$Task = 'default',
    [switch]$Bootstrap,
    [parameter(ParameterSetName = 'Help')]
    [switch]$Help
)

$ErrorActionPreference = 'Stop'
$psakeFile = './psakeFile.ps1'

if ($Bootstrap) {
    # Patch TLS protocols for older Windows versions
    [System.Net.ServicePointManager]::SecurityProtocol = (
        [System.Net.ServicePointManager]::SecurityProtocol -bor
        [System.Net.SecurityProtocolType]::Tls12 -bor
        [System.Net.SecurityProtocolType]::Tls13
    )

    Get-PackageProvider -Name Nuget -ForceBootstrap | Out-Null
    Set-PSRepository -Name PSGallery -InstallationPolicy Trusted

    # Pin PowerShellGet to v2
    $powerShellGetModule = Get-Module -Name 'PowerShellGet' -ListAvailable |
        Where-Object { $_.Version.Major -eq 2 } |
        Sort-Object -Property 'Version' -Descending |
        Select-Object -First 1

    $powerShellGetModuleParameters = @{
        Name           = 'PowerShellGet'
        MinimumVersion = '2.0.0'
        MaximumVersion = '2.99.99'
        Force          = $true
    }

    if (-not $powerShellGetModule) {
        Install-Module @powerShellGetModuleParameters -Scope 'CurrentUser' -AllowClobber
    }
    Import-Module @powerShellGetModuleParameters

    # Register internal repository (idempotent)
    $repositoryName = 'internal-nuget-repo'
    if (-not (Get-PSRepository -Name $repositoryName -ErrorAction 'SilentlyContinue')) {
        $repositoryUrl = "https://nuget.example.com/api/v2/$repositoryName"
        $registerPSRepositorySplat = @{
            Name                      = $repositoryName
            SourceLocation            = $repositoryUrl
            PublishLocation           = $repositoryUrl
            ScriptSourceLocation      = $repositoryUrl
            InstallationPolicy        = 'Trusted'
            PackageManagementProvider = 'NuGet'
        }
        Register-PSRepository @registerPSRepositorySplat
    }

    # Install PSDepend if missing
    if (-not (Get-Module -Name PSDepend -ListAvailable)) {
        Install-Module -Name PSDepend -Repository PSGallery -Scope CurrentUser -Force
    }

    # Try-import-first pattern
    $psDependParameters = @{
        Path          = $PSScriptRoot
        Recurse       = $False
        WarningAction = 'SilentlyContinue'
        Import        = $True
        Force         = $True
        ErrorAction   = 'Stop'
    }

    $importSucceeded = $false
    try {
        Invoke-PSDepend @psDependParameters
        $importSucceeded = $true
        Write-Verbose 'Successfully imported existing modules.' -Verbose
    }
    catch {
        Write-Verbose "Could not import all required modules: $_" -Verbose
        Write-Verbose 'Attempting to install missing or outdated dependencies...' -Verbose
    }

    if (-not $importSucceeded) {
        try {
            Invoke-PSDepend @psDependParameters -Install
        }
        catch {
            Write-Error "Failed to install and import required dependencies: $_"
            Write-Error 'This may be due to locked module files. Please restart the build environment or clear module locks.'
            if ($_.Exception.InnerException) {
                Write-Error "Inner exception: $($_.Exception.InnerException.Message)"
            }
            throw
        }
    }
}
else {
    if (-not (Get-Module -Name 'PSDepend' -ListAvailable)) {
        throw 'Missing dependencies. Please run with the "-Bootstrap" flag to install dependencies.'
    }
    Invoke-PSDepend -Path $PSScriptRoot -Recurse $False -WarningAction 'SilentlyContinue' -Import -Force
}

if ($PSCmdlet.ParameterSetName -eq 'Help') {
    Get-PSakeScriptTasks -buildFile $psakeFile |
        Format-Table -Property Name, Description, Alias, DependsOn
}
else {
    Set-BuildEnvironment -Force
    Invoke-psake -buildFile $psakeFile -taskList $Task -nologo
    exit ([int](-not $psake.build_success))
}
```

## Wrapping Up

The community `build.ps1` pattern gets you 80% of the way — bootstrap,
help, CI exit codes, and build environment detection are all table stakes. The
patterns above handle the remaining 20%: the edge cases that surface when you're
running concurrent CI jobs, onboarding new developers, or pulling dependencies
from internal feeds.

None of these patterns are complex on their own. The value is in combining them
into a single entry point that just works — whether you're running
`.\build.ps1 -Bootstrap` for the first time or kicking off your hundredth CI
build.

For further reading:

- [psake](https://psake.dev) — the build automation tool
- [PowerShellBuild](https://github.com/psake/PowerShellBuild) — common psake
  build tasks for PowerShell modules
- [PSDepend](https://github.com/RamblingCookieMonster/PSDepend) — declarative
  dependency management
