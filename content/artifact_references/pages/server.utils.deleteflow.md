---
title: Server.Utils.DeleteFlow
hidden: true
tags: [Server Artifact]
---

This artifact permanently deletes a flow including it's metadata and
uploaded files.

NOTE: This action can not be undone! The collection is deleted
permanently. Since this is a sensitive operation, typically only
users with the administrator role can run it.


<pre><code class="language-yaml">
name: Server.Utils.DeleteFlow
description: |
  This artifact permanently deletes a flow including it's metadata and
  uploaded files.

  NOTE: This action can not be undone! The collection is deleted
  permanently. Since this is a sensitive operation, typically only
  users with the administrator role can run it.

type: SERVER

required_permissions:
  - MACHINE_STATE

parameters:
  - name: FlowId
    description: The flow ID to delete
    default:
  - name: FlowIds
    description: Delete Multiple Flows
    type: json_array
    default: "[]"
  - name: ClientId
    description: The client id that the collection was done on
    default:
  - name: ReallyDoIt
    description: If you really want to delete the collection, check this.
    type: bool
  - name: Sync
    description: If specified we ensure delete happens immediately
    type: bool

sources:
  - query: |
       LET FlowIds &lt;= if(condition=FlowId, then=FlowIds + FlowId, else=FlowIds)

       SELECT *
       FROM foreach(row={
         SELECT _value AS FlowId FROM foreach(row=FlowIds)
         WHERE log(message="Deleteing Flow " + FlowId, dedup=-1)
       },
       query={
         SELECT Type, Data.VFSPath AS VFSPath, Error
         FROM delete_flow(flow_id=FlowId,
            client_id=ClientId, really_do_it=ReallyDoIt, sync=Sync)
       })

</code></pre>

