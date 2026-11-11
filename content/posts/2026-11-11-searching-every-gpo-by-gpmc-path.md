---
title: "Searching every GPO by the path GPMC shows you (and the DLL hunt it took to get there)"
date: 2026-11-11T09:00:00-07:00
slug: searching-every-gpo-by-gpmc-path
tags: [group-policy, powershell, active-directory, reverse-engineering, security]
description: "I wanted to search all our GPOs by the same Computer Configuration > ... paths that hardening guides use. Those paths aren't in the GPO XML anywhere, so I went digging for the thing in Windows that builds them."
---

Pretty much every hardening guide, CIS benchmark, vendor KB and pentest report tells you to change a GPO the same way:

> Go to **Computer Configuration > Policies > Windows Settings > Security Settings > Account Policies > Kerberos Policy** and set **Maximum lifetime for service ticket** to 600 minutes.

Cool. So I have 2,000 GPOs, which ones set that and what are they set to?

GPMC won't tell you, you just click through GPOs one at a time until you find it or give up. `Get-GPOReport -ReportType Xml` does get you the data, but here's what that exact setting looks like in the XML:

```xml
<q1:Account>
  <q1:Name>MaxServiceAge</q1:Name>
  <q1:SettingNumber>600</q1:SettingNumber>
  <q1:Type>Kerberos</q1:Type>
</q1:Account>
```

There's no "Maximum lifetime for service ticket" in there, no "minutes", and definitely no "Account Policies > Kerberos Policy". The words the guide uses (which are also the words you see in GPMC) **just aren't in the XML**. You can grep it all day and you won't find what the guide is talking about.

And as far as I could find, nothing out there solves this. Every "search all your GPOs" script I came across (Microsoft's own SearchGPOsForSetting script included) text-searches the report XML, so it hits the exact same wall. GPMC's built-in search only matches setting names for some extension types and never values, and the commercial tools I found stick to Administrative Templates. Nothing I found rebuilds the actual GPMC paths, or gets them from the same engine GPMC uses.

So I built a module that fixes that. It indexes every setting in every GPO from a snapshot of the domain, and you can search it by the setting name the way GPMC shows it, by the value (raw or the way GPMC displays it), by GPO name, or by the GPMC path itself. That last one is the part I actually cared about, searching `"Security Options > Domain Controller"` or `"Public Key Policies"` gives you everything under that node across every GPO.

```text
PS> Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'Maximum lifetime for service ticket' |
>>     Format-List GPOName, Scope, GpmcPath, GpmcFields

GPOName    : Default Domain Policy
Scope      : Computer
GpmcPath   : Policies > Windows Settings > Security Settings > Account Policies/Kerberos Policy
GpmcFields : {@{Name=Policy; Value=Maximum lifetime for service ticket}, @{Name=Setting; Value=600 minutes}}
```

*(All the output in this post is mocked up from a lab-style domain, none of it is from a real environment.)*

Code is at [github.com/cartergriffin/gpos](https://github.com/cartergriffin/gpos). I'm going to start with the stuff you can actually do with it, then get into how it works, which is honestly the more interesting part since it took forever to find the piece of Windows that actually builds those paths.

## Stuff you can do with it

Once every setting in the domain has a GPMC path, a name, a raw value and a display value, a lot of annoying questions turn into one-liners. I tried to get something in here for sysadmins, blue team and red team.

Quick primer on what comes back so the queries make sense. Each result is one setting. `GpmcPath` is where it shows up in GPMC, `GpmcFields` is the name/value text GPMC displays for it, and `GpmcPathStatus` says whether the path came out clean (`Ok` most of the time, more on the others further down). Every raw field from the XML is on there too as its own property, like `SettingNumber` or `'EditText/Value'`.

### Sysadmins

Before making a new GPO, check whether something in the domain already does what you want. Say you need to push a trusted root cert out to everything. You google how to do it, the guide says it lives under Public Key Policies > Trusted Root Certification Authorities, and you search for that exact text:

```powershell
# the specific node, i.e. GPOs already pushing trusted roots
Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'Trusted Root Certification Authorities' |
    Group-Object GPOName | Select-Object Count, Name

# the broader node, catches GPOs pushing intermediates, enterprise trust, cert enrollment settings etc
Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'Public Key Policies' |
    Select-Object GPOName, GpmcPath, Item
```

Now you know which GPO the new cert probably belongs in, so you don't end up with yet another one-setting GPO doing the same job. And even if you do make a new one, you've got a working example from your own domain right there instead of fumbling through the config from scratch. This works for pretty much anything a guide hands you a path for.

The one I use most is finding where some old server is still referenced. UNC paths end up hiding in scripts, drive maps, printers, shortcuts, file copies and folder redirection, and one search covers all of them:

```powershell
Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'oldfs01' |
    Select-Object GPOName, Scope, GpmcPath, Item
```

Same idea for startup scripts, this gets you every GPO that has them and what they run.

```powershell
Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'Windows Settings > Scripts > Startup' |
    Select-Object GPOName, Scope, GpmcPath, GpmcFields
```

This is how you find the scripts you were sure you'd removed from the domain, plus the logon script from 20 years ago that's still setting a registry value your shiny new policy is supposed to own. Swap `Startup` for `Shutdown`, `Logon` or `Logoff` to cover the rest. Anything a script writes on every boot is going to fight whatever the policy says, so it's worth checking any script that touches the same keys as your settings.

And if you've ever wondered which GPOs are the big ones.

```powershell
Search-AllGPOSettingsDynamic -JsonPath $jp |
    Where-Object GpmcPathStatus -in 'Ok', 'OkExtensionLevel' |
    Group-Object GPOName | Sort-Object Count -Descending |
    Select-Object -First 20 Count, Name
```

Overlap is the other big one. If you group results by `GpmcPath` and `Item` across GPOs, any group with more than one distinct value is two GPOs disagreeing about the same setting. That's basically every "it works in that OU but not this one" ticket I've ever gotten.

### Blue team

You can check a baseline the way it's actually written. Paste the path straight out of the CIS doc and see who sets it and to what:

```powershell
Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'Security Options > Domain Controller' |
    Select-Object GPOName, Item, @{ n = 'Value'; e = { ($_.GpmcFields | Where-Object Name -eq 'Setting').Value } }
```

Finding "allow everything" values is another good one, like ADMX edit boxes that someone set to `*`:

```text
PS> Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'Allow unsolicited incoming messages' |
>>     Where-Object { $_.'EditText/Value' -eq '*' } |
>>     Select-Object GPOName, GpmcPath, 'EditText/State', 'EditText/Value'

GPOName              GpmcPath                                                                   EditText/State EditText/Value
-------              --------                                                                   -------------- --------------
Lab Workstation Base Policies > Administrative Templates > Network/Network Connections/Windows... Enabled        *
```

For broad principals in sensitive places, search for `Everyone` or `Authenticated Users` and filter `GpmcPath` down to `User Rights Assignment`, `Restricted Groups`, or the file system and registry ACLs.

Certificates are in there too. Every cert pushed through Public Key Policies comes back as an item. The blob is a serialized certificate store element and not a bare DER cert, and property 32 inside it is the actual certificate. Decode that and sort by `NotAfter` and you'll find expired roots that are still getting deployed, and probably a couple intermediates nobody remembers adding.

### Red team

Group Policy Preferences passwords are still a thing. MS14-025 took away the ability to *set* them years ago but it never cleaned up the ones that were already there. `cpassword` is just an attribute, so it gets indexed as a pair name like anything else:

```powershell
Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'cpassword' |
    Select-Object GPOName, Extension, ItemPath, Item
```

The other one is writable stuff that runs as SYSTEM. Startup scripts, scheduled tasks and software installs point at files on shares, and if a non-admin can write to one of those files that's code execution on every machine the GPO applies to. Pull every executed path out of the index and check the ACLs:

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

Check the parent folders too, because if you can replace a file you can usually create one next to it. Taking the snapshot only needs read access to GPOs, which any domain user has by default, so blue teams should probably run this before somebody else does.

You can also just go looking for known-bad configs, since you can search by the same setting names every audit report and hardening guide uses. SMBv1 not being disabled, SMB signing not required, NTLMv1 still allowed via the LAN Manager authentication level, RDP open to way more people than it should be, PowerShell script block logging only turned on for some machines. Search the setting name and look at what each GPO sets it to:

```powershell
'Configure SMBv1 server',
'Microsoft network server: Digitally sign communications (always)',
'LAN Manager authentication level',
'Allow log on through Remote Desktop Services',
'Turn on PowerShell Script Block Logging' | ForEach-Object {
    Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString $_ |
        Select-Object GPOName, Scope, Item, @{ n = 'Value'; e = { ($_.GpmcFields | Where-Object Name -eq 'Setting').Value } }
}
```

Keep in mind that a setting nobody configured won't show up at all, and sometimes that missing result is the finding. Point is, whatever's on your list of bad stuff, you can surface it everywhere at once.

And the big one. That User Rights Assignment search from the blue team section is a good place to start, because it shows who's been handed things like `Debug programs`, `Back up files and directories` or `Allow log on locally` in GPOs that apply to domain controllers. Any of those going to the wrong group on a DC is bad news. But the even bigger problem is the GPO itself. If anyone other than domain admins can **edit** a GPO that applies to the Domain Controllers OU at all, it doesn't really matter what that GPO currently configures. They can add a startup script or an immediate scheduled task to it, every DC will run it as SYSTEM on the next refresh, and that's the domain. The snapshot doesn't capture GPO permissions yet (see the end of the post), so for now this one's a live query with the GroupPolicy module:

```powershell
$dcOu = 'OU=Domain Controllers,' + ([ADSI]'LDAP://RootDSE').defaultNamingContext
foreach ($link in (Get-GPInheritance -Target $dcOu).InheritedGpoLinks) {
    Get-GPPermission -Guid $link.GpoId -All |
        Where-Object { -not $_.Denied -and $_.Permission -in 'GpoEdit', 'GpoEditDeleteModifySecurity' -and
                       $_.Trustee.Name -notin 'Domain Admins', 'Enterprise Admins', 'SYSTEM' } |
        Select-Object @{ n = 'GPO'; e = { $link.DisplayName } }, @{ n = 'Trustee'; e = { $_.Trustee.Name } }, Permission
}
```

`InheritedGpoLinks` includes the domain-level GPOs as well, since those hit DCs too. It doesn't include site-linked GPOs, and it won't expand nested group membership for you, so a harmless-looking group with edit rights still needs a look at who's actually in it.

## Try it

```powershell
git clone https://github.com/cartergriffin/gpos.git
cd gpos
Import-Module .\gpos.psd1
Get-MassiveGPOReportSnapshot -outPath C:\temp\gpo
$jp = (Get-ChildItem C:\temp\gpo\all_gpos-*.json | Sort-Object LastWriteTime | Select-Object -Last 1).FullName
Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'Public Key Policies'
```

You'll need a Windows machine with GPMC / RSAT Group Policy tools installed, since that's what puts `Microsoft.GroupPolicy.Reporting` in the GAC. Windows PowerShell 5.1 is the safest runtime for the path mapping. The first search renders and caches every item so it takes a while, after that it's all cached. `-NoGpmcPath` skips the path mapping if all you want is the raw string search.

That's the useful part. The rest of this is how it works under the hood, and how long it took me to get there.

## Step 1, snapshot everything once

I didn't want to hit AD every time I searched for something, so step one is just taking a snapshot. `Get-GPO -All` for the metadata and `Get-GPOReport -Guid <id> -ReportType Xml` for every GPO, all dumped into one JSON file.

```powershell
Import-Module .\gpos.psd1
Get-MassiveGPOReportSnapshot -outPath C:\temp\gpo
# -> C:\temp\gpo\all_gpos-2026_11_11__09_00_00.json
```

Everything after this runs offline against that file. I'd keep the old snapshots around too, they come in handy later.

## Step 2, stop writing a handler for every setting type

The first version of this was `Find-GPOSetting`, which was a big `if`/`elseif` chain that knew where the strings lived for each kind of setting. Registry policies keep them in one spot, firewall rules in another, etc. Every time I searched for something and got nothing back I'd find some other setting type it didn't handle, and add another branch for it.

In a typical domain the report XML has around 30 extension types. Registry, Security, Public Key, Firewall, Scripts, Printers, Drive Maps, Scheduled Tasks, Audit, Software Restriction, AppLocker, and then a long tail of Preferences types that only show up in like one GPO each. If you write a handler per type you're basically always going to be missing whatever got configured last week.

The version that actually works doesn't have any per-type code at all. It scans the whole snapshot once and, for each extension type, figures out which element paths repeat under a single parent. Those are things like certificates, rules, policies, drive maps, and I call each of those repeating records an "item". If some GPO has only one certificate it still gets split out as an item, because some other GPO had two and that's how it learned the shape. Then each item gets flattened into name/value pairs. Element paths become the names, attributes become `Path@name`, and repeated siblings get `[0]`, `[1]` and so on. Anything left over in an extension gets lumped into one `(extension)` item so nothing gets dropped. The search string gets matched (case-insensitive, literal) against the extension type, the item path, and every pair name and value.

A setting type that didn't exist when I wrote the code gets indexed the same as everything else. The item shapes get cached next to the snapshot so this pass only runs once.

That got me "find any string in any GPO", which was nice, but still no path. And the path was the whole reason I started this.

## Step 3, so where does "Public Key Policies" come from

GPMC's HTML report (`Get-GPOReport -ReportType Html`) shows the full tree, so *something* is turning the XML into "Security Settings > Public Key Policies/Trusted Root Certification Authorities". I could either hand-write a mapping table (I tried, for about an hour, and it was subtly wrong pretty much right away) or go find whatever Windows uses and call it myself.

So I went looking. I'm including all the wrong turns here because that's where most of the useful tricks ended up being.

### Theory 1, it's an XSL stylesheet

XML goes in and an HTML report comes out. Surely that's XSLT, right?

I wrote a quick scanner, `Find-BinaryString`, that reads files in `System32` and `System32\en-US` and searches the raw bytes for `<xsl:stylesheet`, `xsl:template` and the heading text, in both ASCII and UTF-16LE. The UTF-16 part is important because Windows stores most of its strings that way, and plain `Select-String` will skip right past them.

I didn't want to just blindly scan for a stylesheet I wasn't even sure existed though. I wanted to know which binary actually produces the report first, so I went and looked at the code.

### Following the cmdlet down

`Get-GPOReport` lives in the GroupPolicy module, which is managed code, so ILSpy can open it. Small gotcha first. In PowerShell 7 the module loads through the Windows PowerShell compatibility layer, so `(Get-Command Get-GPOReport).DLL` gives you nothing because what you're holding is a proxy function. Ask Windows PowerShell 5.1 instead, and it'll also tell you where the next layer lives:

```powershell
powershell.exe -NoProfile -Command 'Import-Module GroupPolicy; $a = [Microsoft.GroupPolicy.Gpo].Assembly; $a.Location; $a.GetReferencedAssemblies().Name'
```

The cmdlet is really thin. It gets a `Gpo` object, calls `GenerateReport` on it, and that's about it. `Gpo` lives in `Microsoft.GroupPolicy.Management.dll` in the GAC, and this is the entire method.

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

It just passes through to `_GPMGPO`, which is a COM interop type, and its `GenerateReport` is `[MethodImpl(InternalCall)]` with no body. So the report comes out of the GPMC COM server, which is native code in `C:\Windows\System32\gpmgmt.dll`, and ILSpy (correctly) won't open it.

### Taking gpmgmt.dll apart

Fun fact, 7-Zip will open PE files (Portable Executable, the file format Windows uses for .exe and .dll files) as archives (right-click, 7-Zip, Open archive), and you get every section and resource as its own file. `gpmgmt.dll` doesn't have a lot of resources. There's type library data, some COM registration scripts, version info, and an unnamed resource type `169` with two small items in it. `Format-Hex` showed both of those were XML, which was exciting for about four seconds until they turned out to be the backup schemas that `Backup-GPO` uses. I ran the scanner over the extracted `.rdata`, `.text` and `.data` sections and found no stylesheet and no "Public Key Policies".

`.rdata` was still worth it though. Dumping its strings turned up a bunch of debug/log messages that pretty much describe what the code does:

```text
CGPMReport::GenerateReportFromTemplate : GenerateReport failed with 0x%x
<html>This is a template report.<br/>DC:<br/>ID:</html>
CoCreateInstance for ReportFactory ...
GenerateFromGPO / GenerateFromBackup / GenerateFromRSOP
ConvertToXml / ConvertToHtml
```

So `gpmgmt.dll` doesn't build the report either! It `CoCreateInstance`s something called a **ReportFactory**, hands it the GPO, and asks it to convert to XML or HTML. Great, so what is ReportFactory and where does it live?

### A bunch of confident dead ends

This is the part that took about an hour.

I searched the registry for a COM class with "report" in the name and got nothing useful, since the CLSID is hard-coded and has no friendly name. Then I diffed the loaded modules before and after running `Get-GPOReport`. The new ones were `gpmgmt.dll` and its static imports, including `GPRSoP.DLL` and `SCECLI.dll`, so I dumped every string out of both. Turns out `GPRSoP` is the RSoP wizard UI and `scecli` is the Security Settings client-side extension, and neither of them builds reports.

After that I scanned all of System32 for "Public Key Policies". Hits came back in `certmgr.dll` (the Certificates snap-in), `manage-bde.exe.mui`, and `gpresult.exe.mui`. I was pretty sure gpresult was the answer for a while since it produces the same kind of report. It wasn't, the hit was the `gpresult /?` help text lol.

All of that came back empty. About the only thing I learned was where the factory *wasn't*.

### The check I should have done first

Before going any deeper I finally checked that the HTML report even had the headings in it, and how they were marked up:

```html
<div class="he3"><span class="sectionTitle" tabindex="0">Public Key Policies
```

That actually told me a lot. The nesting is encoded in the `heN` class names, and `sectionTitle` is a string that has to exist somewhere in whatever generates the HTML. So I scanned for `sectionTitle` plus the XSL namespace URI (`w3.org/1999/XSL/Transform`, since every stylesheet has to declare it no matter what prefix it uses). That found exactly one Group Policy file, `en-US\gppref.dll.mui`, with three stylesheets inside.

So Theory 1 was right after all... but only for Group Policy Preferences. Preferences items get rendered through XSL, and those stylesheets are nice readable rules. Nothing else uses them though. Security Settings, Administrative Templates and Public Key Policies were still unaccounted for.

### The second type library

At this point the plan was to disassemble `gpmgmt.dll`, which meant getting a copy onto a machine with real analysis tools. You don't actually have to copy a binary off a work machine to do that. Microsoft's public symbol server serves the released binaries too, keyed by the timestamp and image size from the PE header:

```powershell
# On the machine that has the DLL: print the symbol server key (TimeDateStamp + SizeOfImage)
$b  = [IO.File]::ReadAllBytes("$env:SystemRoot\System32\gpmgmt.dll")
$pe = [BitConverter]::ToInt32($b, 0x3C)
'{0:X8}{1:X}' -f [BitConverter]::ToUInt32($b, $pe + 8), [BitConverter]::ToUInt32($b, $pe + 24 + 56)

# Anywhere else:
# https://msdl.microsoft.com/download/symbols/gpmgmt.dll/<that key>/gpmgmt.dll
```

Mine downloaded as a file called `blob`. It's the DLL, just rename it and check the signature with `Get-AuthenticodeSignature`.

Didn't end up needing to disassemble anything. Earlier I'd only pulled strings out of `TYPELIB\1`, which describes the public GPMC COM API that everyone already knows about. But the DLL has a second one, and **`TYPELIB\2` declares the report factory**. And it isn't native!

```text
Class:    Microsoft.GroupPolicy.Reporting.ReportFactory
Assembly: Microsoft.GroupPolicy.Reporting, Version=2.0.0.0, PublicKeyToken=31bf3856ad364e35
IReportFactory: GenerateFromGPO, GenerateFromBackup, GenerateFromRSOP,
                ConvertToXml, ConvertToHtml, InitializeAdm, IncludeComments, ...
```

The report engine is a **.NET assembly** that gets loaded through COM interop. That's why none of my native string scans ever found it, it's not in System32 at all. It lives in the GAC.

```text
C:\Windows\Microsoft.NET\assembly\GAC_MSIL\Microsoft.GroupPolicy.Reporting\v4.0_2.0.0.0__31bf3856ad364e35\Microsoft.GroupPolicy.Reporting.dll
```

About 2.4 MB, plus a 540 KB satellite resources assembly with the English strings in it. And ILSpy opens all of it.

## Step 4, read the engine and then borrow it

Decompiled, the whole thing is just plain C#. `ConvertToHtml` builds the page out of C# template classes (`Microsoft.GroupPolicy.Reporting.Html.Gpo.GpoReportTemplate`, plus a renderer per section), so outside of Preferences there's no XSL anywhere. The display text comes out of the resource table. For example `AccountPolicyRenderer` maps `MaxServiceAge` to "Maximum lifetime for service ticket" with a value format of `{0} minutes`, and every section type has something like that. The XML report turned out to be the same C# object graph run through `XmlSerializer`, which is where those weird `xsi:type="q1:SecuritySettings"` names come from, and why the `q1`/`q2`/`q7` prefixes change from document to document.

The part that really mattered was `GenerateFromBackup(path)`. It reads a report XML file off disk, deserializes it, and gives you back an object you can call `ConvertToHtml()` on, without ever talking to the domain.

My snapshot already has the report XML for every GPO, so I can get the actual GPMC engine to render the actual GPMC HTML for any of it, offline:

```powershell
Add-Type -Path 'C:\Windows\Microsoft.NET\assembly\GAC_MSIL\Microsoft.GroupPolicy.Reporting\v4.0_2.0.0.0__31bf3856ad364e35\Microsoft.GroupPolicy.Reporting.dll'
$tmp = Join-Path $env:TEMP 'one-gpo.xml'
[IO.File]::WriteAllText($tmp, $snapshot[0].Report, [Text.Encoding]::Unicode)   # it expects UTF-16
$html = [Microsoft.GroupPolicy.Reporting.ReportFactory]::new().GenerateFromBackup($tmp).ConvertToHtml()
```

### Turning a render into a path

If you render a whole GPO you get a tree of headings with tables under them, but you still don't know which row belongs to which XML item. You could try matching values between the two, but that gets fuzzy fast and I really didn't want anything fuzzy in this.

What I ended up doing is just giving the engine nothing else to render. For each item I build a copy of the GPO's XML that contains only that one item, render it, and read the chain of `sectionTitle` headings out. Since there's only one thing in the report, that chain *is* the item's GPMC path, and the table rows under it are the exact name/value text GPMC shows. No mapping table, no heuristics, it's deterministic. Identical items get hashed and only rendered once (a root CA pushed by 50 GPOs costs one render), and the results get cached next to the snapshot so only the first run is slow.

The text from the render goes on each result as `GpmcFields`, and that's also how raw values become readable. `SettingNumber = 1` on `LDAPServerIntegrity` doesn't mean anything to me until the engine says it means **None**:

```text
PS> $r = Search-AllGPOSettingsDynamic -JsonPath $jp -SearchString 'LDAP server signing requirements'
PS> $r | Format-List GPOName, GpmcPath, SettingNumber, GpmcFields

GPOName       : Default Domain Controllers Policy
GpmcPath      : Policies > Windows Settings > Security Settings > Local Policies/Security Options > Domain Controller
SettingNumber : 1
GpmcFields    : {@{Name=Policy; Value=Domain controller: LDAP server signing requirements}, @{Name=Setting; Value=None}}
```

### Stuff that broke along the way

Some of the failures were more interesting than the actual fixes, so here they are.

Some templates expect siblings. The wireless (WLan) renderer dereferences both the profile list and the network filter, so rendering one item by itself throws a null reference. When a single-item render fails I now render the whole extension and use the heading chain all of its rows share. Those results get marked `OkExtensionLevel` so you can tell them apart.

Audit policy needed MMC, which I didn't see coming. Every `AuditSettings` render failed with `Could not load file or assembly 'Microsoft.ManagementConsole, Version=3.0.0.0'`. The assembly was on the box, but it was in the *legacy* GAC (`C:\Windows\assembly`), which the .NET 4 runtime doesn't look in. GPMC finds it fine, PowerShell doesn't. An `AssemblyResolve` handler that looks in the old GAC by name, version and public key token fixed every Audit item in one go.

GPMC hides disabled preference items. `PolicyMakerSettingsTemplate` skips anything where `IsDisabled()` is true, so those render to nothing. They're still in the GPO and you still want to be able to find them, so I flip `disabled="1"` to `0` on the throwaway copy, render it, and mark the result `OkDisabled`.

Some things really just aren't shown. Leftover `(extension)` fields and registry entries with no value (key-only deletes, for example) come back as `NotRendered`. Every item gets a status, so nothing silently disappears and you can see the gaps.

The runtime matters too. `Microsoft.GroupPolicy.Reporting` is .NET Framework code. It mostly works loaded into PowerShell 7 on modern .NET, but I didn't want to trust "mostly", so the repo has `tests/Compare-GPORuntimes.ps1`. It runs the same snapshot under both shells with separate caches and diffs every path and status. I treat Windows PowerShell 5.1 as the reference since that's the runtime GPMC itself uses.

And PowerShell was the slow part. Doing the splitting, flattening and matching per item in PowerShell took about 9 seconds per search, and loading plus parsing the snapshot was only half a second of that. Moving the indexing into C# (`private/GpoSnapshotIndex.cs`, compiled with `Add-Type` when the module loads) made it fast enough to actually use interactively.

## Compared to GPOZaurr

[GPOZaurr](https://github.com/EvotecIT/GPOZaurr) is great and you should run it. It's built for GPO hygiene stuff like duplicate (CNF) objects, broken and disabled links, empty GPOs, ownership and permission problems, and AD vs SYSVOL consistency. Basically it tells you whether your Group Policy infrastructure is healthy.

It does look inside the settings too, its GPOAnalysis report sorts everything in your GPOs into 70+ categories using its own parsing. What it doesn't do is give you GPMC's actual paths and display text, or let you search by them. That's the gap this fills, what each GPO configures in the words GPMC (and every hardening guide) uses, and where else that same thing is configured. I'd run both.

## Stuff I'd tell myself before starting

If the text isn't in the XML then something else is adding it, and it's way less work to find that thing and call it than to rebuild its rules by hand. Scan binaries in UTF-16LE too, since that's how Windows stores most of its strings. 7-Zip opens PE files, and the log strings in `.rdata` describe the code better than most of the docs do. Just because something's a native COM server doesn't mean it's native all the way down, so read every type library in the binary and not just the first one. Microsoft's symbol server hands out binaries as well as symbols, which is handy when you're not supposed to be copying files off a box. And check the output before digging into the internals. One regex on the HTML (`sectionTitle`) found more than an hour of DLL scanning did.

I did most of this with Claude as a pair. It was fast at writing scanners and reading decompiled code, and it was also confidently wrong about where the engine lived more than once. Most of the progress came from not accepting "it's probably not possible" and running one more check. If you find a setting type that comes back `NotRendered` or `Error`, open an issue, that status is there so failures show up instead of getting quietly dropped.

## What would make this better (PRs welcome)

The biggest missing piece is permissions. Right now the index knows what every GPO configures, but not who can change it, and that's most of the red team value left on the table. The report XML already has a `SecurityDescriptor` section with the trustee permissions for each GPO, and a `LinksTo` section with every OU, domain or site it's linked to (plus whether the link is enabled or enforced). So a lot of this is plumbing, getting those onto each result next to `GpmcPath` and `GpmcFields`.

Once that's in, the DC check above becomes one offline query. It also opens up the more general version of it, any GPO a non-admin can edit that runs code anywhere. That means startup scripts, scheduled tasks, software installs, services, or just a registry value that points at a binary, on any workstation or server, with the scope worked out from the links. Security filtering and WMI filters would round that out, so "linked to the OU" turns into "actually applies to these machines".

A few other ACLs would be worth pulling in too. There's who can write `gPLink` on each OU (anyone who can link an existing GPO to the Domain Controllers OU is effectively in the same spot as someone who can edit one), the SYSVOL folder ACLs behind each GPO, since those files are what clients actually read, and owners of the GPO objects themselves. If any of that sounds fun, the repo's open and I'd love the help.
