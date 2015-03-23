---
layout: default
title: Creating Enterprise Versions of Public Chocolatey Packages
---

# Creating Enterprise Versions of Public Chocolatey Packages

In this post I address the issue of Chocolatey packages containing external links to msi, exe and other files. 

## The Problems

### Package Review
For a large enterprise, packages must go through a review process before installation. 
As part of this review process, external links are discouraged because they require external calls during install and can make restoration of service difficult when the links change. 

### External Dependencies
In order to effectively utilize Chocolatey in the enterprise, packages that contain external links need to be rewritten with the external dependencies linked to an internal folder share or embedded in the package. 
The approach taken will depend on the size of the installer and how frequently updates to the chocolatey package are needed. 
Because Chocolatey packages are immutable, special care should be taken to releasing frequent iterations with an embedded solution. 
A large installer of, say, 1GB embedded in a package will quickly eat up space in a storage medium.
In these cases, it may be better to rewrite the package to utilize a internal link to the installer instead of embedding the installer.

## The anatomy of a Chocolatey package

### What is a .nupkg file?
A chocolatey package is nothing more than a zip file with a .nupkg extension. 
Using a tool like [7zip](https://chocolatey.org/packages/7zip.Install) you can explore the file contents as you would any other zip file.

I'll be using 7zip to explore the package contents, but you could easily use a tool like Nuget Package Explorer or rename the file with a .zip extension and use the built in windows explorer zip support.

### Examining the internals of a Chocolatey package

For this post, I'll be taking the chocolatey package for Debug Diag and updating it to be both a self contained package and a package with an internal location for the installer. 
The link for the Debug Diag chocolatey package is located here: 

[debug diag](https://chocolatey.org/packages/debugdiagnostic)

### Using 7z.exe on the Command Line

In order to run 7z.exe runs 7zip from the command line, you may have to add the install directory to the environment variable path before 7z will execute on the command prompt.

```
PS > 7z l .\debugdiagnostic.2.1.0.7.nupkg

7-Zip [64] 9.22 beta  Copyright (c) 1999-2011 Igor Pavlov  2011-04-18

Listing archive: .\debugdiagnostic.2.1.0.7.nupkg

--
Path = .\debugdiagnostic.2.1.0.7.nupkg
Type = zip
Physical Size = 3931

   Date      Time    Attr         Size   Compressed  Name
------------------- ----- ------------ ------------  ------------------------
2015-01-20 10:14:24 .....          500          500  _rels\.rels
2015-01-20 10:14:24 .....         1830          786  debugdiagnostic.nuspec
2015-01-20 10:14:24 .....          265          156  tools\chocolateyInstall.ps1
2015-01-20 10:14:24 .....         1179         1179  package\services\metadata\core-properties\2838bbf8051044378d3fbfab8a2ea205.psmdcp
2015-01-20 10:14:24 .....          448          448  [Content_Types].xml
------------------- ----- ------------ ------------  ------------------------
                                  4222         3069  5 files, 0 folders
```

The "_rels" and "package" directories are created by chocolatey when using the choco pack operation. 

## Creating a New Download Location

We will be updating the tools\chocolateyInstall.ps1 file in order to update the installer download location.

### Extracting

Extracting the chocolateyInstall.ps1 file using 7zip we have the following command.

```
7z x debugdiagnostic.2.1.0.7.nupkg -odebugdiagnostic.2.1.0.7 -r
```

This command will extract the package to a folder called "debugdiagnostic.2.1.0.7" to the file system.
We can see the files produced by running the DIR command on the directory where the .nupkg lives.

```
dir debugdiagnostic.2.1.0.7 /b

debugdiagnostic.nuspec
package
tools
[Content_Types].xml
_rels
```

Opening the tools\chocolateyInstall.ps1 file we see the following contents

```powershell
Install-ChocolateyPackage 'debugdiagnostic' 'msi' '/quiet' 'http://download.microsoft.com/download/B/4/6/B46E9984-5DF2-4B56-AE32-D60A88C2A6D8/DebugDiagx86.msi' 'http://download.microsoft.com/download/B/4/6/B46E9984-5DF2-4B56-AE32-D60A88C2A6D8/DebugDiagx64.msi'
```

We expand this script to several lines so that it is easier to see what is going on.

```powershell
$packageName = 'debugdiagnostic'
$fileType = 'msi'
$silentArgs = '/quiet'
$url = 'http://download.microsoft.com/download/B/4/6/B46E9984-5DF2-4B56-AE32-D60A88C2A6D8/DebugDiagx86.msi'
$url64bit = 'http://download.microsoft.com/download/B/4/6/B46E9984-5DF2-4B56-AE32-D60A88C2A6D8/DebugDiagx64.msi'
Install-ChocolateyPackage $packageName $fileType $silentArgs $url $url64bit
```

Given this information, we now see that the chocolatey package is pulling the installer from two Microsoft hosted links. 
Lets change these links to be a UNC share on our local network. 

### Rewrite

Open the extracted ChocolateyInstall.ps1 and paste the locations to the UNC shares of http urls of the installers. 
Save the file and close the text editor.

Example:

```powershell
$packageName = 'debugdiagnostic'
$fileType = 'msi'
$silentArgs = '/quiet'
$url = '\\InstallerHost\Microsoft\DebugDiag\2.1.0.7\DebugDiagx86.msi'
$url64bit = '\\InstallerHost\Microsoft\DebugDiag\2.1.0.7\DebugDiagx64.msi'
Install-ChocolateyPackage $packageName $fileType $silentArgs $url $url64bit
```

### Update

Now we need to take the original chocolatey package and update the chocolateyInstall.ps1 with the file we just edited.

We can use the "7z a" command to re-create the package. 
We will use the extension .zip to create the file for now and we will rename in the next section.

```
7z a debugdiagnostic.2.1.0.7.zip .\debugdiagnostic.2.1.0.7\* -r

7-Zip [64] 9.20  Copyright (c) 1999-2010 Igor Pavlov  2010-11-18
Scanning

Creating archive debugdiagnostic.2.1.0.7.zip

Compressing  debugdiagnostic.nuspec
Compressing  package\services\metadata\core-properties\2838bbf8051044378d3fbfab8a2ea205.psmdcp
Compressing  tools\chocolateyInstall.ps1
Compressing  [Content_Types].xml
Compressing  _rels\.rels

Everything is Ok
```

So now we have a debugdiagnostic.2.1.0.7.zip file and a debugdiagnostic.2.1.0.7.nupkg file. 
We will rename the existing "nupkg" file to a "nupkg.bak" extension and the "zip" file to "nupkg" extension.

```
move debugdiagnostic.2.1.0.7.nupkg debugdiagnostic.2.1.0.7.nupkg.bak
move debugdiagnostic.2.1.0.7.zip debugdiagnostic.2.1.0.7.nupkg
```

Running DIR on the directory we now see the two files

```
dir /b *.nupkg.*
debugdiagnostic.2.1.0.7.nupkg
debugdiagnostic.2.1.0.7.nupkg.bak
```

### Summary 

We can now take the debugdiagnostic.2.1.0.7.nupkg and post it to a internal repository. 

## Creating a Self Contained Package
