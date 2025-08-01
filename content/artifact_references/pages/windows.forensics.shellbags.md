---
title: Windows.Forensics.Shellbags
hidden: true
tags: [Client Artifact]
---

Windows uses the Shellbag keys to store user preferences for GUI
folder display within Windows Explorer.

This artifact uses the raw registry parser to inspect various user
registry hives around the filesystem for BagMRU keys. Different OS
versions may have slightly different locations for the MRU keys.


<pre><code class="language-yaml">
name: Windows.Forensics.Shellbags
description: |
  Windows uses the Shellbag keys to store user preferences for GUI
  folder display within Windows Explorer.

  This artifact uses the raw registry parser to inspect various user
  registry hives around the filesystem for BagMRU keys. Different OS
  versions may have slightly different locations for the MRU keys.

reference:
  - https://www.sans.org/blog/computer-forensic-artifacts-windows-7-shellbags/

parameters:
  - name: SearchSpecs
    type: csv
    description: Define locations of MRU bags in various registries.
    default: |
      HiveGlob,KeyGlob
      C:/Users/*/NTUSER.dat,\Software\Microsoft\Windows\Shell\BagMRU\**
      C:/Users/*/AppData/Local/Microsoft/Windows/UsrClass.dat,\Local Settings\Software\Microsoft\Windows\Shell\BagMRU\**


imports:
  # Link files use the same internal format as shellbags so we import
  # the profile here.
  - Windows.Forensics.Lnk

sources:
  - query: |
      LET AllHives = SELECT * FROM foreach(row=SearchSpecs,
        query={
            SELECT OSPath AS HivePath, KeyGlob
            FROM glob(globs=HiveGlob)
            WHERE log(message="Inspecting hive " + HivePath)
        })

      LET ShellValues = SELECT * FROM foreach(row=AllHives,
        query={
           SELECT OSPath, Data, ModTime
           FROM glob(
              root=pathspec(DelegatePath=HivePath),
              globs=KeyGlob,
              accessor="raw_reg")
           WHERE Data.type =~ "BINARY" AND OSPath.Basename =~ "^[0-9]+$"
        })

      LET ParsedValues = SELECT
          OSPath.Dirname AS KeyPath, OSPath.Basename AS ValueName,
          parse_binary(profile=Profile, filename=Data.value,
                       accessor="data", struct="ItemIDList") as _Parsed,
          base64encode(string=Data.value) AS _RawData, ModTime
      FROM ShellValues

      LET AllResults &lt;= SELECT KeyPath, ValueName,
        _Parsed.ShellBag.Description AS Description,
        _Parsed, _RawData, ModTime
      FROM ParsedValues

      // Recursive function to join path components together.
      // Limit recursion depth just in case.
      LET FormPath(MRUPath, Description, Depth) = SELECT * FROM if(
        condition=Depth &lt; 20,
        then={SELECT * FROM chain(
          b={
            SELECT MRUPath, Description, Depth,
              -- Signify unknown component as ?
              Description.LongName || Description.ShortName || "?" AS Name
            FROM scope()
          },
          c={
            SELECT * FROM foreach(row={
              SELECT KeyPath, ValueName, Description, Depth
              FROM AllResults
              WHERE KeyPath = MRUPath.Dirname
              LIMIT 1
            }, query={
              SELECT * FROM FormPath(MRUPath=KeyPath,
                                     Description=Description, Depth=Depth + 1)
            })
          })
          ORDER BY Depth DESC
          LIMIT 10
        })

        // Now display all hits and their reconstructed path
        LET ReconstructedPath = SELECT ModTime, KeyPath, ValueName, Description, {
           SELECT * FROM FormPath(
                MRUPath=KeyPath, Description=Description, Depth=0)
        } AS Chain, _RawData, _Parsed
        FROM AllResults

        SELECT ModTime,
           KeyPath AS _OSPath,
           KeyPath.DelegatePath AS Hive,
           KeyPath.Path AS KeyPath, ValueName, Description,
               join(array=Chain.Name, sep=" -&gt; ") AS Path,
               _RawData, _Parsed
        FROM ReconstructedPath
        ORDER BY Path

column_types:
  - name: _RawData
    type: base64

</code></pre>

