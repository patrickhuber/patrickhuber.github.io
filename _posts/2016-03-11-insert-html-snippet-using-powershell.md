---
layout: default
title: Insert Html Snippet into Existing Html Document with Powershell
---
I recently needed to insert a html snippet into the beginning of a html document using powershell and found a regex solution that 
did the job. 

```powershell
$html = "
<html>
    <head></head>
    <body some-attribute=`"`"></body>
</html>"

$htmlSplit = $html -split "(<body[^>]+>)"
$htmlSplit[1]+= "
<div>
    <p>this is a paragraph</p>
</div>"
[string]::Join("", $htmlSplit)
```
