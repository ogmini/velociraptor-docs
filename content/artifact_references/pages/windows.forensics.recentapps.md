---
title: Windows.Forensics.RecentApps
hidden: true
tags: [Client Artifact]
---

GUI Program execution launched on the Win10 system is tracked in the
RecentApps key.

NOTE: This artifact is available up from Windows 10 1607 to 1709.
After that, the RecentApps key is no longer populated in the
referenced location. Previously existing data is not removed.

DEPRECATION: This artifact is deprecated and will be removed
soon. It is replaced by the registry hunter.


<pre><code class="language-yaml">
name: Windows.Forensics.RecentApps
description: |
  GUI Program execution launched on the Win10 system is tracked in the
  RecentApps key.

  NOTE: This artifact is available up from Windows 10 1607 to 1709.
  After that, the RecentApps key is no longer populated in the
  referenced location. Previously existing data is not removed.

  DEPRECATION: This artifact is deprecated and will be removed
  soon. It is replaced by the registry hunter.

reference:
  - https://www.sans.org/security-resources/posters/windows-forensics-evidence-of/75/download
  - https://www.forensicfocus.com/forums/general/forensics-windows-registry-program-launch-history/
  - https://thinkdfir.com/2020/10/23/when-did-recentapps-go/

precondition: SELECT OS From info() where OS = 'windows'

parameters:
  - name: UserFilter
    default: ""
    description: If specified we filter by this user ID.
    type: regex

  - name: ExecutionTimeAfter
    default: ""
    type: timestamp
    description: If specified only show executions after this time.

  - name: RecentAppsKey
    default: Software\Microsoft\Windows\CurrentVersion\Search\RecentApps\*

  - name: UserHomes
    default: C:\Users\*\NTUSER.DAT

sources:
  - query: |
      LET TMP = SELECT * FROM foreach(
         row={
            SELECT OSPath AS HivePath FROM glob(globs=UserHomes)
         },
         query={
            SELECT timestamp(winfiletime=LastAccessedTime) AS LastExecution,
                   timestamp(winfiletime=LastAccessedTime).Unix AS _LastExecutionTS,
                   Key.OSPath as _OSPath,
                   HivePath[2] AS User,
                    AppId, AppPath, LaunchCount
            FROM read_reg_key(
               globs=RecentAppsKey,
               root=pathspec(
                 DelegateAccessor="ntfs",
                 DelegatePath=HivePath),
               accessor="raw_reg")
         })

      LET A1 = SELECT * FROM if(
          condition=UserFilter,
          then={
            SELECT * FROM TMP WHERE User =~ UserFilter
          }, else={ SELECT * FROM TMP})

      SELECT * FROM if(
          condition=ExecutionTimeAfter,
          then={
            SELECT * FROM A1 WHERE LastExecutionTS &gt; ExecutionTimeAfter
          }, else={ SELECT * FROM A1})

</code></pre>

