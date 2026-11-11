---
title: "Searching every GPO by the path GPMC shows you (and the DLL hunt it took to get there)"
date: 2026-11-11T09:00:00-07:00
slug: searching-every-gpo-by-gpmc-path
tags: [group-policy, powershell, active-directory, reverse-engineering, security]
description: "A PowerShell module that indexes every setting in every GPO in a domain and lets you search by setting name, value, or the exact GPMC path a hardening guide tells you to go to. Plus the reverse engineering it took to rebuild those paths."
---

Every hardening guide, CIS benchmark, vendor KB and pentest report tells you to change a GPO the same way:

> Go to **Computer Configuration > Policies > Windows Settings > Security Settings > Account Policies > Kerberos Policy** and set **Maximum lifetime for service ticket** to 600 minutes.

So you have 800 GPOs. Which of them set that, and to what?

GPMC can't answer that. You click through GPOs one at a time. `Get-GPOReport -ReportType Xml` gets you the data, but this is what that setting looks like in it:

```xml
<q1:Account>
  <q1:Name>MaxServiceAge</q1:Name>
  <q1:SettingNumber>600</q1:SettingNumber>
  <q1:Type>Kerberos</q1:Type>
</q1:Account>
```

It has no "Maximum lifetime for service ticket", no "minutes", and no "Account Policies > Kerberos Policy". The words in the guide, which are also the words you see in GPMC, **are not in the XML at all**. You can grep it all day and never find what the guide is talking about.

I built a module that fixes this. It indexes every setting in every GPO in a domain snapshot and lets you search by:

- the setting name, as GPMC shows it,
- the value, raw or as GPMC displays it,
- the GPO name,
- **or the GPMC path itself**, so `"Security Options > Domain Controller"` or `"Public Key Policies"` returns everything under that node in every GPO.

```text
PS> Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'Maximum lifetime for service ticket' |
>>     Format-List GPOName, Scope, GpmcPath, GpmcFields

GPOName    : Default Domain Policy
Scope      : Computer
GpmcPath   : Policies > Windows Settings > Security Settings > Account Policies/Kerberos Policy
GpmcFields : {@{Name=Policy; Value=Maximum lifetime for service ticket}, @{Name=Setting; Value=600 minutes}}
```

*(All output in this post is mocked up from a lab-style domain. None of it comes from a real environment.)*

The code is at [github.com/cartergriffin/gpos](https://github.com/cartergriffin/gpos). The rest of this post covers how it works, because getting those paths meant finding the piece of Windows that builds them, and that took a while.

## Step 1: Snapshot everything once

Hitting AD every time you want to search is slow and rude, so the first step is a snapshot: `Get-GPO -All` for metadata, plus `Get-GPOReport -Guid <id> -ReportType Xml` for each GPO, all dumped into one JSON file.

```powershell
Import-Module .\gpos.psd1
Get-MassiveGPOReportSnapshot -outPath C:\temp\gpo
# -> C:\temp\gpo\all_gpos-2026_11_11__09_00_00.json
```

Everything after this runs offline against that file. Keep the old snapshots, because you'll want them later.

## Step 2: Stop writing a handler per setting type

The first version of this tool was `Find-GPOSetting`: a big `if`/`elseif` chain that knew where strings live in each kind of setting. Registry policies keep them here, firewall rules there, and so on. Every time I searched for something and got nothing back, I found another setting type it didn't handle and added a branch.

The report XML has around 30 extension types in a typical domain: Registry, Security, Public Key, Firewall, Scripts, Printers, Drive Maps, Scheduled Tasks, Audit, Software Restriction, AppLocker, and a long tail of Preferences types that show up in one GPO each. Writing a handler per type means you're always one new type behind.

The version that works has no per-type code:

1. **Learn the item shapes from the data.** Scan the whole snapshot once. For each extension type, find the element paths that repeat under one parent: certificates, rules, policies, drive maps. Those repeating records are "items". A GPO that has only one certificate still gets it split out, because some other GPO had two.
2. **Flatten each item into name/value pairs.** Element paths become names, attributes become `Path@name`, and repeated siblings get `[0]`, `[1]`. Whatever is left over in an extension becomes a single `(extension)` item, so nothing is dropped.
3. **Match the search string against everything.** That covers the extension type, the item path, every pair name and every pair value. It's case-insensitive and literal.

A setting type that didn't exist when I wrote the code gets indexed the same way as the rest. The item shapes are cached next to the snapshot, so this pass runs once.

That got me "find any string in any GPO". It didn't get me the path, and the path was the part I actually wanted.

## Step 3: Where does "Public Key Policies" come from?

GPMC's HTML report (`Get-GPOReport -ReportType Html`) shows the full tree, so something turns XML into "Security Settings > Public Key Policies/Trusted Root Certification Authorities". My options were to hand-write a mapping table (I tried, briefly, and it was wrong in subtle ways almost immediately) or to find whatever Windows uses and call it.

I went looking. Here's how that went, wrong turns included, because the wrong turns are where the useful techniques are.

### Theory 1: it's an XSL stylesheet

An XML document becomes an HTML report. That has to be XSLT, right?

I wrote a quick scanner, `Find-BinaryString`, that reads files in `System32` and `System32\en-US` and searches the raw bytes for `<xsl:stylesheet`, `xsl:template` and the heading text, in both ASCII and UTF-16LE. The UTF-16 part matters, because Windows stores most of its strings that way and a plain `Select-String` skips right past them.

Scanning blindly for a stylesheet I wasn't sure existed seemed premature, though. First I wanted to know which binary actually produces the report, so I went to look at the code.

### Following the cmdlet down

`Get-GPOReport` lives in the GroupPolicy module, which is managed code, so ILSpy can open it. One gotcha first: in PowerShell 7 the module loads through the Windows PowerShell compatibility layer, so `(Get-Command Get-GPOReport).DLL` returns nothing because you're holding a proxy function. Ask Windows PowerShell 5.1 instead, which also tells you where the next layer lives:

```powershell
powershell.exe -NoProfile -Command 'Import-Module GroupPolicy; $a = [Microsoft.GroupPolicy.Gpo].Assembly; $a.Location; $a.GetReferencedAssemblies().Name'
```

The cmdlet is thin. It gets a `Gpo` object and calls `GenerateReport`, and that's it. `Gpo` lives in `Microsoft.GroupPolicy.Management.dll` in the GAC, and here is the whole method:

```csharp
public string GenerateReport(ReportType reportType, out GPStatusMessageCollection statusMessages)
{
    object pvarGPMProgress = null;
    object pvarGPMCancel = null;
    GPMResult gPMResult = _GPMGPO.GenerateReport((GPMReportType)reportType, ref pvarGPMProgress, out pvarGPMCancel);
    statusMessages = new GPStatusMessageCollection(gPMResult.Status);
    gPMResult.OverallStatus();
    return (string)gPMResult.Result;
}
```

It's a pass-through to `_GPMGPO`, a COM interop type whose `GenerateReport` is `[MethodImpl(InternalCall)]` with no body. So the report comes from the GPMC COM server, which is the native `C:\Windows\System32\gpmgmt.dll`. ILSpy correctly refuses to open that.

### Taking gpmgmt.dll apart

7-Zip opens PE files as archives (right-click, 7-Zip, Open archive), which gives you every section and resource as a file. `gpmgmt.dll` has very few resources: type library data, some COM registration scripts, version info, and an unnamed resource type `169` with two small items. `Format-Hex` showed both were XML. Exciting for about four seconds, until they turned out to be the backup schemas `Backup-GPO` uses. Running the scanner over the extracted `.rdata`, `.text` and `.data` sections found no stylesheet and no "Public Key Policies".

`.rdata` was still useful, though. Dumping its strings turned up debug and log messages that describe what the code does:

```text
CGPMReport::GenerateReportFromTemplate : GenerateReport failed with 0x%x
<html>This is a template report.<br/>DC:<br/>ID:</html>
CoCreateInstance for ReportFactory ...
GenerateFromGPO / GenerateFromBackup / GenerateFromRSOP
ConvertToXml / ConvertToHtml
```

So `gpmgmt.dll` doesn't build the report either. It `CoCreateInstance`s a **ReportFactory**, hands it the GPO, and asks it to convert to XML or HTML. Great: what is ReportFactory, and where does it live?

### A series of confident dead ends

This is the part that took about an hour:

- **Registry search for a COM class named "report":** nothing relevant. The CLSID is hard-coded and has no friendly name.
- **Diffing loaded modules before and after `Get-GPOReport`:** the new modules were `gpmgmt.dll` and its static imports, `GPRSoP.DLL` and `SCECLI.dll` among them. I dumped every string from both. `GPRSoP` is the RSoP wizard UI and `scecli` is the Security Settings client-side extension. Neither one builds reports.
- **Scanning all of System32 for "Public Key Policies":** hits in `certmgr.dll` (the Certificates snap-in), `manage-bde.exe.mui`, and `gpresult.exe.mui`. That last one looked like the answer for a while, since gpresult produces the same kind of report. It wasn't. The hit was `gpresult /?` help text.

Every one of these came back empty, which mostly told me what the factory wasn't.

### The check I should have run first

Before digging further, I confirmed the HTML report contains the headings at all, and looked at how they're marked up:

```html
<div class="he3"><span class="sectionTitle" tabindex="0">Public Key Policies
```

That gave me two things. The nesting is in the `heN` class names, and `sectionTitle` is a string that has to exist wherever the HTML is generated. Scanning for `sectionTitle` plus the XSL namespace URI (`w3.org/1999/XSL/Transform`, which every stylesheet has to declare whatever prefix it uses) found exactly one Group Policy file: `en-US\gppref.dll.mui`, with three stylesheets inside it.

So Theory 1 was right, but only for Group Policy Preferences. Preferences items render through XSL, and those stylesheets are clean, readable rules. Nothing else does. Security Settings, Administrative Templates and Public Key Policies weren't covered.

### The second type library

At this point the plan was to disassemble `gpmgmt.dll`, which meant getting a copy onto a machine with real analysis tools. You don't have to copy a binary off a work machine for that. Microsoft's public symbol server serves released binaries, keyed by the PE header's timestamp and image size:

```powershell
# On the machine that has the DLL: print the symbol server key (TimeDateStamp + SizeOfImage)
$b  = [IO.File]::ReadAllBytes("$env:SystemRoot\System32\gpmgmt.dll")
$pe = [BitConverter]::ToInt32($b, 0x3C)
'{0:X8}{1:X}' -f [BitConverter]::ToUInt32($b, $pe + 8), [BitConverter]::ToUInt32($b, $pe + 24 + 56)

# Anywhere else:
# https://msdl.microsoft.com/download/symbols/gpmgmt.dll/<that key>/gpmgmt.dll
```

Mine came down as a file called `blob`. It's the DLL, so rename it and check the signature with `Get-AuthenticodeSignature`.

It never needed disassembling. Earlier I'd only read strings out of `TYPELIB\1`, which describes the public GPMC COM API everyone knows. The DLL has a second one. **`TYPELIB\2` declares the report factory**, and it isn't native:

```text
Class:    Microsoft.GroupPolicy.Reporting.ReportFactory
Assembly: Microsoft.GroupPolicy.Reporting, Version=2.0.0.0, PublicKeyToken=31bf3856ad364e35
IReportFactory: GenerateFromGPO, GenerateFromBackup, GenerateFromRSOP,
                ConvertToXml, ConvertToHtml, InitializeAdm, IncludeComments, ...
```

The report engine is a **.NET assembly**, loaded through COM interop. That's why none of the native string scans found it: it isn't in System32. It's here:

```text
C:\Windows\Microsoft.NET\assembly\GAC_MSIL\Microsoft.GroupPolicy.Reporting\v4.0_2.0.0.0__31bf3856ad364e35\Microsoft.GroupPolicy.Reporting.dll
```

It's about 2.4 MB, with a 540 KB satellite resources assembly holding the English strings. ILSpy opens all of it.

## Step 4: Read the engine, then borrow it

Decompiled, the whole thing is plain C#:

- `ConvertToHtml` builds the page from C# template classes (`Microsoft.GroupPolicy.Reporting.Html.Gpo.GpoReportTemplate` and one renderer per section). There's no XSL outside Preferences.
- Display text comes from the resource table. For example, `AccountPolicyRenderer` maps `MaxServiceAge` to "Maximum lifetime for service ticket" with a value format of `{0} minutes`. Every section type has an equivalent.
- The XML report is the same C# object graph run through `XmlSerializer`. That's where the odd `xsi:type="q1:SecuritySettings"` names come from, and why the `q1`/`q2`/`q7` prefixes change from document to document.
- **`GenerateFromBackup(path)` reads a report XML file from disk, deserializes it, and gives you an object you can call `ConvertToHtml()` on.** It never talks to the domain.

That last point was the one that mattered. The snapshot already holds report XML for every GPO, so the actual GPMC engine can render GPMC's actual HTML for any of it, offline:

```powershell
Add-Type -Path 'C:\Windows\Microsoft.NET\assembly\GAC_MSIL\Microsoft.GroupPolicy.Reporting\v4.0_2.0.0.0__31bf3856ad364e35\Microsoft.GroupPolicy.Reporting.dll'
$tmp = Join-Path $env:TEMP 'one-gpo.xml'
[IO.File]::WriteAllText($tmp, $snapshot[0].Report, [Text.Encoding]::Unicode)   # it expects UTF-16
$html = [Microsoft.GroupPolicy.Reporting.ReportFactory]::new().GenerateFromBackup($tmp).ConvertToHtml()
```

### Turning a render into a path

Rendering a whole GPO gives you a tree of headings with tables underneath, but which row belongs to which XML item? You could try matching values between the two, but that's fuzzy, and nothing about this problem should be fuzzy.

The trick is to give the engine nothing else to render. For each item:

1. Build a copy of the GPO's XML that contains **only that one item**.
2. Render it.
3. Read the chain of `sectionTitle` headings. With only one thing in the report, that chain *is* the item's GPMC path, and the table rows under it are the exact name/value text GPMC shows.

No mapping table and no heuristics, so the result is deterministic. Identical items are hashed and rendered once (a root CA pushed by 50 GPOs costs one render), and results are cached next to the snapshot, so only the first run is slow.

The text from the render goes on each result as `GpmcFields`, which is also what makes raw values readable. `SettingNumber = 1` on `LDAPServerIntegrity` is meaningless until the engine says it means **None**:

```text
PS> $r = Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'LDAP server signing requirements'
PS> $r | Format-List GPOName, GpmcPath, SettingNumber, GpmcFields

GPOName       : Default Domain Controllers Policy
GpmcPath      : Policies > Windows Settings > Security Settings > Local Policies/Security Options > Domain Controller
SettingNumber : 1
GpmcFields    : {@{Name=Policy; Value=Domain controller: LDAP server signing requirements}, @{Name=Setting; Value=None}}
```

### Things that broke along the way

A few failures had more interesting causes than the bugs themselves, so here they are:

- **Some templates expect siblings.** The wireless (WLan) renderer dereferences both the profile list and the network filter, so a one-item render throws a null reference. The fix: when a single-item render fails, render the whole extension and use the heading chain all of its rows share. Those results are marked `OkExtensionLevel`, so you can tell them apart.
- **Audit policy needed MMC.** Every `AuditSettings` render failed with `Could not load file or assembly 'Microsoft.ManagementConsole, Version=3.0.0.0'`. The assembly was there, but in the *legacy* GAC (`C:\Windows\assembly`), which the .NET 4 runtime doesn't search. GPMC finds it and PowerShell doesn't. An `AssemblyResolve` handler that looks in the old GAC by name, version and public key token fixed every Audit item at once.
- **GPMC hides disabled preference items.** `PolicyMakerSettingsTemplate` skips any item where `IsDisabled()` is true, so they render to nothing. They're still in the GPO, and you still want to find them. The fix is to flip `disabled="1"` to `0` on the throwaway copy, render it, and mark the result `OkDisabled`.
- **Some things really aren't shown.** Leftover `(extension)` fields and registry entries with no value (key-only deletes, for example) come back `NotRendered`. Nothing is silently dropped. Every item gets a status, so gaps are visible.
- **Runtime matters.** `Microsoft.GroupPolicy.Reporting` is .NET Framework code. It mostly works when loaded into PowerShell 7 on modern .NET, but "mostly" isn't good enough for a tool you trust, so the repo has `tests/Compare-GPORuntimes.ps1`. It runs the same snapshot under both shells with separate caches and diffs every path and status. Windows PowerShell 5.1 is the reference, since it's the runtime GPMC itself uses.
- **PowerShell was the slow part.** Splitting, flattening and matching per item in PowerShell took about 9 seconds per search, and loading plus parsing the snapshot was only half a second of that. Moving the indexing into C# (`private/GpoSnapshotIndex.cs`, compiled with `Add-Type` when the module loads) made it fast enough to use interactively.

## What you can do with it

Once every setting in the domain has a GPMC path, a name, a raw value and a display value, a lot of questions become one-liners. There's something here for everyone.

### Sysadmins

**"Where is that old server still referenced?"** UNC paths hide in scripts, drive maps, printers, shortcuts, file copies and folder redirection. One search covers all of them:

```powershell
Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'oldfs01' |
    Select-Object GPOName, Scope, GpmcPath, Item
```

**"Which GPOs touch Scripts, and what do they run?"**

```powershell
Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'Windows Settings > Scripts > Startup' |
    Select-Object GPOName, Scope, GpmcPath, GpmcFields
```

**"Which GPOs are the big ones?"**

```powershell
Search-AllGPOSettingsDynamic -JsonPath $jp |
    Where-Object GpmcPathStatus -in 'Ok', 'OkExtensionLevel' |
    Group-Object GPOName | Sort-Object Count -Descending |
    Select-Object -First 20 Count, Name
```

**Overlap.** Group results by `GpmcPath` and `Item` across GPOs, and any group with more than one distinct value is two GPOs disagreeing about the same setting. That's the classic "it works in that OU but not this one" ticket.

### Blue team

**Check a baseline the way it's written.** Paste the path from the CIS doc and see who sets it and to what:

```powershell
Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'Security Options > Domain Controller' |
    Select-Object GPOName, Item, @{ n = 'Value'; e = { ($_.GpmcFields | Where-Object Name -eq 'Setting').Value } }
```

**Find "allow everything" values.** For example, ADMX edit boxes set to `*`:

```text
PS> Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'Allow unsolicited incoming messages' |
>>     Where-Object { $_.'EditText/Value' -eq '*' } |
>>     Select-Object GPOName, GpmcPath, 'EditText/State', 'EditText/Value'

GPOName              GpmcPath                                                                   EditText/State EditText/Value
-------              --------                                                                   -------------- --------------
Lab Workstation Base Policies > Administrative Templates > Network/Network Connections/Windows... Enabled        *
```

**Broad principals in sensitive places.** Search for `Everyone` or `Authenticated Users` and filter `GpmcPath` to `User Rights Assignment`, `Restricted Groups`, or file system and registry ACLs.

**Certificates.** Every cert pushed by Public Key Policies comes back as an item. The blob is a serialized certificate-store element rather than a bare DER cert, and property 32 inside it is the certificate. Decode it and sort by `NotAfter` to find expired roots that are still being deployed, and intermediates nobody remembers adding.

### Red team

**Group Policy Preferences passwords.** MS14-025 removed the ability to *set* them years ago, but it didn't clean up the ones already there. The `cpassword` attribute is an indexed pair name like any other:

```powershell
Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'cpassword' |
    Select-Object GPOName, Extension, ItemPath, Item
```

**Writable things that run as SYSTEM.** Startup scripts, scheduled tasks and software installs point at files on shares. If a non-admin can write to one of those files, that's code execution on every machine the GPO applies to. Pull every executed path out of the index and check its ACL:

```powershell
$exec = Search-AllGPOSettingsDynamic -JsonPath $jp |
    Where-Object Extension -in 'Scripts', 'ScheduledTasksSettings', 'SoftwareInstallationSettings' |
    ForEach-Object { $_.GpmcFields.Value } |
    Select-String -Pattern '\\\\[^\s"]+\.(exe|ps1|bat|cmd|vbs|msi)' -AllMatches |
    ForEach-Object { $_.Matches.Value } | Sort-Object -Unique

foreach ($p in $exec) {
    (Get-Acl -LiteralPath $p).Access |
        Where-Object { $_.FileSystemRights -match 'Write|Modify|FullControl' -and
                       $_.IdentityReference -notmatch 'SYSTEM|Administrators|Domain Admins|Enterprise Admins|CREATOR OWNER' } |
        Select-Object @{ n = 'Path'; e = { $p } }, IdentityReference, FileSystemRights
}
```

Check the parent folders too: if you can replace a file, you can usually also create one next to it. The snapshot only needs read access to GPOs, which any domain user has by default, so blue teams should run this before someone else does.

## How this compares to GPOZaurr

[GPOZaurr](https://github.com/EvotecIT/GPOZaurr) is great, and you should run it. It's built for GPO *hygiene*: duplicate (CNF) objects, broken and disabled links, empty GPOs, ownership and permission problems, AD vs. SYSVOL consistency. It answers "is my Group Policy infrastructure healthy?"

This tool answers a different question: "what does each GPO actually configure, in the words GPMC uses, and where else is the same thing configured?" It searches and compares at the setting level, which as far as I can tell GPOZaurr doesn't do. They complement each other well.

## Try it

```powershell
git clone https://github.com/cartergriffin/gpos.git
cd gpos
Import-Module .\gpos.psd1
Get-MassiveGPOReportSnapshot -outPath C:\temp\gpo
$jp = (Get-ChildItem C:\temp\gpo\all_gpos-*.json | Sort-Object LastWriteTime | Select-Object -Last 1).FullName
Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'Public Key Policies'
```

Requirements: a Windows machine with GPMC / RSAT Group Policy tools, which is what puts `Microsoft.GroupPolicy.Reporting` in the GAC. Windows PowerShell 5.1 is the safest runtime for the path mapping. The first search renders and caches every item, so it takes a while. After that, searches use the cache. `-NoGpmcPath` skips path mapping if you just want the raw string search.

## Takeaways

- **If an XML file doesn't contain the text, something else adds it.** Find that thing and call it, rather than rebuilding its rules by hand.
- **Scan binaries in UTF-16LE too.** Most Windows strings are stored that way.
- **7-Zip opens PE files**, and log strings in `.rdata` describe the code better than most docs.
- **"Native COM server" doesn't mean native all the way down.** Read every type library in the binary, not just the first one.
- **Microsoft's symbol server hands out binaries too.** You don't need to copy files off a box you shouldn't copy files off.
- **Check the output before you dig into the internals.** One regex on the HTML (`sectionTitle`) found more than an hour of scanning DLLs did.

I did most of this with Claude as a pair. It was fast at writing scanners and reading decompiled code, and it was also confidently wrong about where the engine lived more than once. The breakthroughs came from refusing to accept "it's probably not possible" and running one more check. If you find a setting type that comes back `NotRendered` or `Error`, open an issue. That status exists so failures show up instead of disappearing.
