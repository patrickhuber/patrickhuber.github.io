---
layout: default
title: Creating Reusable PowerShell Modules with PsGet and Chocolatey
---
# Creating Reusable PowerShell Modules with PsGet and Chocolatey

As automation becomes more popular, teams need a way to organize scripts in source control and ultimately deliver those scripts to a runtime environment in a consistent and repeatable way. 

## The Problems

### Install
I have a few PowerShell modules that I work with that I'd like to distribute internally but I don't want to manually copy the modules into the PowerShell modules folders. 

### Dependency Management
I would also like to benefit from package management and not need to copy folders and files around in order to get reuse out of my modules. If a module has a dependency, it should automatically download that dependency.

### Learning Curve
I would like to give the Operations team a one liner that they can use to install the latest package dependencies and not have to worry about digging around on google to find out how to install the module. 

### I want it now
Windows Management Framework v5.0 is currently in Preview and has package management capabilities that I can't use because they are not production ready. 

## The Solutions
### NuGet - Dependency Management

NuGet allows for an application to set a dependency on a released package. If the package has dependencies, those dependencies are downloaded automatically when the package is resolved. 
The ability to resolve packages solves the issue of managing PowerShell dependencies. 
NuGet out of the box is great, but it is really meant for managing development dependencies and requires some extra extension to manage software. 

This leads us to...

### Chocolatey - Software as Packages

Built upon NuGet, [chocolatey](https://chocolatey.org/) takes the concept of packages and applies it to software installs. 
This gives the consumer the ability to script the install of software and not just development artifacts.  

So the install of something like Java is reduced to something like:

```
cinst jre8
```

### PsGet - Automate Install of PowerShell Modules

But what about PowerShell modules? PowerShell modules aren't installed in the traditional sense, they are really copied to a folder in the user's PowerShell Modules directory. 
Enter [PsGet](http://psget.net/). PsGet allows you to install a module by pointing to a .psm1 file. 

### Putting it all together

We now have individual solutions for all of our problems above, so lets put it all together and build a Chocolatey package for our IO module. 

## Directory Structure

### Source Control

I store my powershell modules in a single Git repo using subfolders for each module. You could easily modify this and store a module per git repo. 

### On Disk

For this example I'm going to create a Chocolatey package called "IO" that has a Cmdlet called "Join-Uri". 
The Join-Uri cmdlet allows me to combine a Uri with a child path and works very similar to the Join-Path cmdlet. 

The folder you create here should match the module name that you want to create.

I will start by creating a folder called "IO" and placing a PowerShell script called Join-Uri.ps1 in the folder. 
Running DIR on the folder I see:

```
Directory: \IO

Mode                LastWriteTime     Length Name                                                                                                       
----                -------------     ------ ----                                                                                                
-a---         3/17/2015  11:29 AM        492 Join-Uri.ps1  
```

Here are the contents of the Join-Uri.ps1 script

```powershell
<# Joins uri to a child path#>
function Join-Uri
{
    [CmdletBinding(DefaultParametersetName="Uri")]    
    param(
        [Parameter(ParameterSetName="Uri", Mandatory=$true, Position=0)]
        [uri]$uri, 
        [Parameter(ParameterSetName="Uri", Mandatory=$true, Position=1)]
        [string]$childPath)
    $combinedPath = [system.io.path]::Combine($uri.AbsoluteUri, $childPath)
    $combinedPath = $combinedPath.Replace('\', '/')
    return New-Object uri $combinedPath
}
```

Pretty simple. It just takes a path and combines it using System.IO.Path.Combine and rewrites the backslashes to forward slashes. 

Any other cmdlets that I need to add will be added as individual .ps1 files. 
In order to support this structure, we need to create a PowerShell module file .psm1 that understands how to read individual script files. 
We will create a psm1 files called "IO.psm1" in the directory. 
The file name should match the name of the module you want to create and because of that it should be the same as the directory that contains it.
EX: folder "IO" holds "IO.psm1".

Here is the output of the DIR command. 

```
Directory: \IO

Mode                LastWriteTime     Length Name                                                                                                       
----                -------------     ------ ----                                                                                                      
-a---         3/17/2015   3:52 PM        171 IO.psm1                                                                                                    
-a---         3/17/2015  11:29 AM        492 Join-Uri.ps1   
```

Here are the contents of the IO.psm1 file

```powershell
# Load all script files recursively into this module
# http://www.kmerwin.com/?p=174
gci $psscriptroot\*.ps1 -exclude ChocolateyInstall.ps1 -Recurse | % {. $_.FullName }
```

Special shout out to the blog at [kmerwin.com](http://www.kmerwin.com/?p=174) for originating this idea for organizing files. 
I added an exclude filter in there to make sure I don't include the ChocolateyInstall.ps1 file in actual module. 
The ChocolateyInstall.ps1 file is the logic that installs our chocolatey package.
We will add that file now.

First, create a tools folder. In the tools folder add a ChocolateyInstall.ps1 file. Running DIR on the root folder will show:

```
Directory: \IO

Mode                LastWriteTime     Length Name                                                                                                       
----                -------------     ------ ----                                                                                                       
d----         3/17/2015   4:02 PM            tools                                                                                                      
-a---         3/17/2015   3:52 PM        171 IO.psm1                                                                                                    
-a---         3/17/2015  11:29 AM        492 Join-Uri.ps1                                                                                               

```

Running DIR on the tools folder now shows:

```
Directory: \IO\tools

Mode                LastWriteTime     Length Name                                               
----                -------------     ------ ----                                               
-a---         3/17/2015   5:23 PM        172 ChocolateyInstall.ps1       
```

Here are the contents of the ChocolateyInstall.ps1 file

```powershell
Import-Module PsGet
$scriptDirectory = $PSScriptRoot
$packageDirectory = ( $scriptDirectory | Split-Path -Parent )
Install-Module -ModulePath "$packageDirectory\IO.psm1"
```

Going line by line:

1. We need to import the PsGet module to get access tot he Install-Module cmdlet. 
2. We are setting the script directory variable by looking at the $PSScriptRoot. The $PSScriptRoot variable was added in PowerShell v3. If you want to use a earlier version of powershell, you can use $myInvocation.MyCommand.Definition.
3. Because the module is located in the package root and the ChocolateyInstall.ps1 file is located in the tools directory, we need to get the parent directory to the ChocolateyInstall.ps1 file.
4. We use the Package Directory to load our IO.psm1 module file using the Install-Module cmdlet from the PsGet module. 

Chocolatey requires a [nuspec file](http://docs.nuget.org/create/nuspec-reference) in order to create a package. 
The NuSpec file is an xml configuration file that contains metadata about the package including the name, version and description. 

Here are the contents of the nuget spec file. 

```xml
<?xml version="1.0" encoding="utf-8"?>
<package xmlns="http://schemas.microsoft.com/packaging/2010/07/nuspec.xsd">
  <metadata>
    <id>IO</id>
    <title>IO Powershell Package</title>
    <version>1.0.0</version>
    <authors>Patrick Huber</authors>
    <owners>Patrick Huber</owners>
    <summary>A package that provides base IO functions not supplied by normal System.IO.</summary>
    <description>A package that provides base IO functions not supplied by normal System.IO</description>        
    <copyright>Patrick Huber 2015</copyright>    
    <requireLicenseAcceptance>false</requireLicenseAcceptance>
	<dependencies>
      <dependency id="PsGet" />
    </dependencies>
  </metadata>
  <files>
	<file src="tools\ChocolateyInstall.ps1" target="tools"/>
    <file src="*.ps1"/>
    <file src="*.psm1"/>
    <file src="*.psd1"/>
  </files>
</package>
```

The nuspec file contents will vary based on the package you are creating. 

* The id is the name of the package you are creating. 
* The version is currently 1.0.0, but you will need to make sure and increment the version as you make updates to your package.  Take note of this version number, we will use it again.
* There is a dependency to PsGet in the "dependencies" section. This tells chocolatey to install the PsGet module before installing this package. 
* The files section lists the ChocolateyInstall.ps1 file and the target is the tools directory in the package
* The files section also includes any file with the extensions (.ps1, .psm1 and .psd1)

We haven't touched on the .psd1 file yet, but it is the last part needed to successfully create a powershell module. 

In the module development directory, we will run the "New-ModuleManifest" command to create the psd1 file.

```powershell
New-ModuleManifest -Path IO.psd1
```

This command generates a generic module manifest file and saves it to IO.psd1. 
Using DIR on the folder, we now see the following files:

```
Directory: C:\src\github\powershell\IO

Mode                LastWriteTime     Length Name                                               
----                -------------     ------ ----                                               
d----         3/17/2015   4:02 PM            tools                                              
-a---         3/17/2015   5:22 PM       1079 IO.nuspec                                          
-a---         3/17/2015   3:56 PM       5218 IO.psd1                                            
-a---         3/17/2015   3:52 PM        171 IO.psm1                                            
-a---         3/17/2015  11:29 AM        492 Join-Uri.ps1     
```

Here are the contents of the IO.psd1 file:

```powershell
#
# Module manifest for module 'IO'
#
# Generated by: Patrick Huber
#
# Generated on: 3/17/2015
#

@{

# Script module or binary module file associated with this manifest.
# RootModule = ''

# Version number of this module.
ModuleVersion = '1.0.0'

# ID used to uniquely identify this module
GUID = '1280e9bd-ec78-4803-bdee-0aa47e08571b'

# Author of this module
Author = 'Patrick Huber'

# Company or vendor of this module
CompanyName = 'Patrick Huber'

# Copyright statement for this module
Copyright = '(c) 2015 Patrick Huber. All rights reserved.'

# Description of the functionality provided by this module
# Description = ''

# Minimum version of the Windows PowerShell engine required by this module
# PowerShellVersion = ''

# Name of the Windows PowerShell host required by this module
# PowerShellHostName = ''

# Minimum version of the Windows PowerShell host required by this module
# PowerShellHostVersion = ''

# Minimum version of the .NET Framework required by this module
# DotNetFrameworkVersion = ''

# Minimum version of the common language runtime (CLR) required by this module
# CLRVersion = ''

# Processor architecture (None, X86, Amd64) required by this module
# ProcessorArchitecture = ''

# Modules that must be imported into the global environment prior to importing this module
# RequiredModules = @()

# Assemblies that must be loaded prior to importing this module
# RequiredAssemblies = @()

# Script files (.ps1) that are run in the caller's environment prior to importing this module.
# ScriptsToProcess = @()

# Type files (.ps1xml) to be loaded when importing this module
# TypesToProcess = @()

# Format files (.ps1xml) to be loaded when importing this module
# FormatsToProcess = @()

# Modules to import as nested modules of the module specified in RootModule/ModuleToProcess
# NestedModules = @()

# Functions to export from this module
FunctionsToExport = '*'

# Cmdlets to export from this module
CmdletsToExport = '*'

# Variables to export from this module
VariablesToExport = '*'

# Aliases to export from this module
AliasesToExport = '*'

# List of all modules packaged with this module.
# ModuleList = @()

# List of all files packaged with this module
# FileList = @()

# Private data to pass to the module specified in RootModule/ModuleToProcess
# PrivateData = ''

# HelpInfo URI of this module
# HelpInfoURI = ''

# Default prefix for commands exported from this module. Override the default prefix using Import-Module -Prefix.
# DefaultCommandPrefix = ''

}
```

The file keeps most of the generated defaults which allows you to add files to the module without much additional editing. 
Any line with a '#' in the front is a comment and will not be processed in the manifest. 
I did change the version number to match the version number in the nuspec file. 

It is important to update both the psd1 version number and the nuspec version number when the version changes in order to maintain consistency. 

## Building the Module

The module structure has now been created and we need to use Chocolatey to package up the module into a nupkg file so we can consume it from the chocolatey feed. 

### Create the Chocolatey Package

In the same directory we have been working, open up a type the following in a powershell prompt:

```
choco pack
```

This command will generate a .nupkg file that has the package name and version number as part of its structure. 
For the example we created above, the file is named IO.1.0.0.nupkg. 

### Move the Chocolatey Package to the Local Feed 

For this example, I will not be publishing the package I create to the public chocolatey feed. 
You can use a file share or a local nuget server for this step if you like. 
I will instead be using a local directory to store my packages and using the chocolatey repository to resolve public package dependencies.

Because I don't want to check the package into source control, I'm going to move it to a feed directory for consumption. 
This feed directory can be anything, but I'll place mine in the root of c:\

```powershell 
mkdir "C:\chocolatey\local-repo"
```

Running DIR on this folder shows my package file.

```
Directory: C:\chocolatey\local-repo

Mode                LastWriteTime     Length Name                                               
----                -------------     ------ ----                                               
-a---         3/17/2015   5:22 PM       4973 IO.1.0.0.nupkg
```

### Installing the Module

At this point the package is ready for consumption. 
The final step will be to consume the package using the "choco install" command. 
We have two repositories to use because we are consuming the PsGet package from the public repository and the IO package from our local repository. 

```
choco install IO -source '"C:\chocolatey\local-repo;https://chocolatey.org/api/v2/"' -force
```

The -source flag is important because it allows us to specify multiple repositories for resolving our packages. 
You need to wrap the packages sources with single quotes ' followed by double quotes ". 
The repositories must also be separated by a semicolon.
I'm only specifying two repositories here, but you could easily specify more.

After we run the command we can now use the Join-Uri cmdlet. 
If using PowerShell v3 and above, the module resolution will occur automatically. 
If using a lower version of powershell, you will need to type "Import-Module IO"

```
# Import Module is optional if using PowerShell v3 and above
# Import-Module IO
Join-Uri "http://www.google.com" "childpath"
```

This returns a URI object. Here is the output from the module.

```
AbsolutePath   : /childpath
AbsoluteUri    : http://www.google.com/childpath
LocalPath      : /childpath
Authority      : www.google.com
HostNameType   : Dns
IsDefaultPort  : True
IsFile         : False
IsLoopback     : False
PathAndQuery   : /childpath
Segments       : {/, childpath}
IsUnc          : False
Host           : www.google.com
Port           : 80
Query          : 
Fragment       : 
Scheme         : http
OriginalString : http://www.google.com/childpath
DnsSafeHost    : www.google.com
IsAbsoluteUri  : True
UserEscaped    : False
UserInfo       : 
```

## Next Steps

From here we can now go and create a set of modules that promote reuse. 
Adding the modules as dependencies through chocolatey we can easily distribute the reusable scripts in a consistent and repeatable way. 

Here is the source for the example I created above.

[IO Powershell Module and Chocolatey Package](https://github.com/patrickhuber/Powershell/tree/master/IO)