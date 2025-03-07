---
title: Windows.Detection.Thumbdrives.OfficeMacros
hidden: true
tags: [Client Event Artifact]
---

Users inserting Thumb drives or other Removable drive pose a
constant security risk. The external drive may contain malware or
other undesirable content. Additionally thumb drives are an easy way
for users to exfiltrate documents.

This artifact watches for any removable drives and scans any added
office documents for VBA macros.

We exclude very large removable drives since they might have too
many files.


```yaml
name: Windows.Detection.Thumbdrives.OfficeMacros
description: |
  Users inserting Thumb drives or other Removable drive pose a
  constant security risk. The external drive may contain malware or
  other undesirable content. Additionally thumb drives are an easy way
  for users to exfiltrate documents.

  This artifact watches for any removable drives and scans any added
  office documents for VBA macros.

  We exclude very large removable drives since they might have too
  many files.

type: CLIENT_EVENT

parameters:
  - name: officeExtensions
    default: "\\.(xls|xlsm|doc|docx|ppt|pptm)$"
    type: regex

sources:
  - query: |
        SELECT * FROM foreach(
          row = {
            SELECT * FROM Artifact.Windows.Detection.Thumbdrives.List()
            WHERE OSPath =~ officeExtensions
          },
          query = {
            SELECT * from olevba(file=OSPath)
          })

```
