---
title: MacOS.System.TCC
hidden: true
tags: [Client Artifact]
---

This artifact provides details around the TCC (Transparency,
Consent, and Control) database, and can help reveal when access to
system services has been added or modified for an application.

Note that this artifact has only been tested on macOS Big Sur, and
that the `allowed`, and `prompt_count` columns will need to be used
in place of the `auth_value`, `auth_reason`, and `auth_version`
columns for Catalina and prior.


```yaml
name: MacOS.System.TCC
description: |
   This artifact provides details around the TCC (Transparency,
   Consent, and Control) database, and can help reveal when access to
   system services has been added or modified for an application.

   Note that this artifact has only been tested on macOS Big Sur, and
   that the `allowed`, and `prompt_count` columns will need to be used
   in place of the `auth_value`, `auth_reason`, and `auth_version`
   columns for Catalina and prior.

type: CLIENT

author: Wes Lambert - @therealwlambert

parameters:
- name: TCCGlob
  default: /Library/Application Support/com.apple.TCC/TCC.db,/Users/*/Library/Application Support/com.apple.TCC/TCC.db

precondition:
      SELECT OS From info() where OS = 'darwin'

sources:
  - query: |
      LET TCCList = SELECT OSPath
       FROM glob(globs=split(string=TCCGlob, sep=","))

      LET TCCAccess = SELECT *
       FROM sqlite(file=OSPath, query="SELECT * from access")

      LET TCCAccessDetails =
          SELECT * FROM foreach(
              row=TCCAccess,
              query={ SELECT
                    timestamp(epoch=last_modified) AS LastModified,
                    service AS Service,
                    client AS Client,
                    if(condition= client_type= 0, then="Console", else=if(condition= client_type= 1, then="Service/Script", else="Other")) AS ClientType,
                    if(condition= auth_value= 2, then="Yes", else="No") AS Allowed,
                    if(condition= OSPath =~ "Users", then=path_split(path=OSPath)[-5], else="System") AS User,
                    auth_reason AS _AuthReason,
                    auth_version AS _AuthVersion,
                    csreq AS _CSReq,
                    policy_id as _PolicyId,
                    indirect_object_identifier_type as _IndirectObjectIdentifierType,
                    indirect_object_identifier as IndirectObjectIdentifier,
                    indirect_object_code_identity as _IndirectObjectCodeIdentity,
                    flags as _Flags,
                    OSPath AS _OSPath
                 FROM scope()
              }
          )
      SELECT * FROM foreach(row=TCCList, query=TCCAccessDetails)

```
