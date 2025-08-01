---
title: Admin.Client.Upgrade.Debian
hidden: true
tags: [Client Artifact]
---

Remotely push new client updates to Debian hosts.

NOTE: This artifact requires that you supply a client Debian package using the
tools interface or using the "debian client" command. Simply click on the tool
in the GUI and upload a package.


<pre><code class="language-yaml">
name: Admin.Client.Upgrade.Debian
description: |
  Remotely push new client updates to Debian hosts.

  NOTE: This artifact requires that you supply a client Debian package using the
  tools interface or using the "debian client" command. Simply click on the tool
  in the GUI and upload a package.

tools:
  - name: VelociraptorDebian

parameters:
  - name: SleepDuration
    default: "600"
    type: int
    description: |
      The package is typically large and we do not want to
      overwhelm the server so we stagger the download over this many
      seconds.

  - name: ServiceName
    default: "velociraptor_client"
    type: str
    description: |
      The name of the service to restart after the upgrade.

implied_permissions:
  - EXECVE
  - FILESYSTEM_WRITE

sources:
  - precondition:
      SELECT OS From info() where OS =~ 'linux'

    query:  |
      // FetchBinary downloads to /tmp on linux
      LET bin &lt;= SELECT OSPath AS Dest
      FROM Artifact.Generic.Utils.FetchBinary(
         ToolName="VelociraptorDebian", IsExecutable=FALSE,
         SleepDuration=SleepDuration)

      // Version handling for older clients.
      LET Rm(X) = if(
        condition=version(function='rm')!=NULL,
        then=rm(filename=X),
        else={ SELECT * FROM execve(argv=["rm", "-f", X]) })

      // Call the binary and return all its output in a single row.
      // If we fail to download the binary we do not run the command.
      SELECT * FROM foreach(row=bin,
      query={
        SELECT * FROM chain(
          // Remove the existing prerm - Previous versions had a bug that
          // would shutdown the service during uninstall. See #3122
          a={SELECT * FROM Rm(X="/var/lib/dpkg/info/velociraptor-client.prerm")},

          // Install the new client
          b={SELECT * FROM execve(argv=["dpkg", "-i", str(str=Dest)])},

          // Restart the client
          c={SELECT * FROM execve(argv=["systemctl", "restart", ServiceName])}
        )
      })

</code></pre>

