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
If you are looking for a pure powershell option and have .NET 4.5 and Powershell 3, [look here](http://ss64.com/ps/zip.html).
The powershell community extensions (pscx) also has an option that you can install via chocolatey [here](https://chocolatey.org/packages/pscx)

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

Extracting the contents of the nupkg file using 7zip we have the following command.

```
7z x debugdiagnostic.2.1.0.7.nupkg -odebugdiagnostic.2.1.0.7 * -r
```

This command will extract the package to a folder called "debugdiagnostic.2.1.0.7" to the file system.
We can see the files produced by running the DIR command on the debugdiagnostic.2.1.0.7 directory.

```
PS \debugdiagnostic.2.1.0.7> dir

Mode                LastWriteTime     Length Name
----                -------------     ------ ----
d----         3/24/2015   9:53 AM            package
d----         3/24/2015   9:56 AM            tools
d----         3/24/2015   9:53 AM            _rels
-----         1/20/2015   9:14 AM       1830 debugdiagnostic.nuspec
-----         1/20/2015   9:14 AM        448 [Content_Types].xml
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

Now we need to recreate the original chocolatey package with the updated chocolateyInstall.ps1.

We can use the "choco pack" command to repackage the artifacts, but we first need to cleanup some of the directories generated during the zip process.

```powershell
rmdir _rels \s \q
rmdir package \s \q
del [Content_Types].xml
```

The folder is ready to be packaged. I ran "choco pack" from the same directory that the .nuspec file exists.

```powershell
choco pack

Chocolatey v0.9.9.1
Attempting to build package from 'debugdiagnostic.nuspec'.
Successfully created package 'debugdiagnostic.2.1.0.7.nupkg'
```

I now need to move the file to the repository directory.

```powershell
move debugdiagnostic.2.1.0.7.nupkg c:\chocolatey\local-repo\
```

### Summary 

We can now take the debugdiagnostic.2.1.0.7.nupkg and post it to a internal repository. 

## Creating a Self Contained Package

We will be updating the tools\chocolateyInstall.ps1 file, pointing it to a self contained installer.

### Extracting 

Similar to the previous section, we first need to extract the contents of the nupkg file to a local folder.

Assume that we have downloaded the package directly from source without performing any modifications.

```
7z x debugdiagnostic.2.1.0.7.nupkg -odebugdiagnostic.2.1.0.7 * -r
```

Again, this command will extract the package to a folder called "debugdiagnostic.2.1.0.7" to the file system.
We can see the files produced by running the DIR command on the directory where the .nupkg lives.

```powershell
PS \debugdiagnostic.2.1.0.7> dir

Mode                LastWriteTime     Length Name
----                -------------     ------ ----
d----         3/24/2015   9:53 AM            package
d----         3/24/2015   9:56 AM            tools
d----         3/24/2015   9:53 AM            _rels
-----         1/20/2015   9:14 AM       1830 debugdiagnostic.nuspec
-----         1/20/2015   9:14 AM        448 [Content_Types].xml
```

### Rewrite

We will go to the download locations in the tools\chocolateyInstall.ps1 and pull down the x86 and x64 installers.

Inspecting the chocolateyInstall.ps1 file, we see the following urls. 

```powershell
'http://download.microsoft.com/download/B/4/6/B46E9984-5DF2-4B56-AE32-D60A88C2A6D8/DebugDiagx86.msi'
'http://download.microsoft.com/download/B/4/6/B46E9984-5DF2-4B56-AE32-D60A88C2A6D8/DebugDiagx64.msi'
```

I'm going to place the installers in the debugdiagnostic.2.1.0.7 folder in the tools directory.

```powershell
$url = 'http://download.microsoft.com/download/B/4/6/B46E9984-5DF2-4B56-AE32-D60A88C2A6D8/DebugDiagx86.msi'
$url64bit = 'http://download.microsoft.com/download/B/4/6/B46E9984-5DF2-4B56-AE32-D60A88C2A6D8/DebugDiagx64.msi'

Invoke-WebRequest $url -OutFile 'DebugDiagx86.msi'
Invoke-WebRequest $url64bit -OutFile 'DebuDiagx64.msi'
```

Running the DIR command on the tools directory we now see the following contents

```powershell
PS \debugdiagnostic.2.1.0.7\tools> dir

Mode                LastWriteTime     Length Name
----                -------------     ------ ----
-----         1/20/2015   9:14 AM        265 chocolateyInstall.ps1
-a---         3/24/2015   9:56 AM   22421504 DebugDiagx64.msi
-a---         3/24/2015   9:56 AM   17412096 DebugDiagx86.msi
```

Now we can modify the tools\chocolateyInstall.ps1 file to use these local msi files instead of using the remote locations.

```powershell
$directory = $PSScriptRoot
$packageName = 'debugdiagnostic'
$fileType = 'msi'
$silentArgs = '/quiet'
$url = Join-Path $directory 'DebugDiagx86.msi'
$url64bit = Join-Path $directory 'DebugDiagx64.msi'
Install-ChocolateyPackage $packageName $fileType $silentArgs $url $url64bit
```

### Update

Similar to the previous example, we will cleanup the directory removing chocolatey artifacts from the base directory.

```powershell
rmdir _rels \s \q
rmdir package \s \q
del [Content_Types].xml
```

The folder is ready to be packaged. Running "choco pack"

```powershell
choco pack

Chocolatey v0.9.9.1
Attempting to build package from 'debugdiagnostic.nuspec'.
Successfully created package 'debugdiagnostic.2.1.0.7.nupkg'
```

Again we move the file to the local repository.

```powershell
move debugdiagnostic.2.1.0.7.nupkg c:\chocolatey\local-repo\
```

### Summary

Again the package is ready to be consumed by choco install.

```powershell
choco install debugdiagnostic -source '"c:\chocolatey\local-repo\"'
```

## Conclusions

We have explored two ways to modify public chocolatey packages to enable enterprise consumption. 
When consuming these packages you will want to make sure to use a local repository in your chocolatey script. 
Also, if dependencies are loaded as part of your chocolatey package, you will need to make sure and modify them as well.