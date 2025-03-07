---
title: Windows.Events.Kerberoasting
hidden: true
tags: [Client Event Artifact]
---

**Description**:
This Artifact will monitor all successful Kerberos TGS Ticket events for
Service Accounts (SPN attribute) implemented with weak encryption. These
tickets are vulnerable to brute force attack and this event is an indicator
of a Kerberoasting attack.

**ATT&CK**: [T1208 - Kerberoasting](https://attack.mitre.org/techniques/T1208/)
Typical attacker methodology is to firstly request accounts in the domain
with SPN attributes, then request an insecure TGS ticket for brute forcing.
This attack is particularly effective as any domain credentials can be used
to implement the attack and service accounts often have elevated privileges.
Kerberoasting can be used for privilege escalation or persistence by adding a
SPN attribute to an unexpected account.

**Reference**: [The Art of Detecting Kerberoast Attacks](https://www.trustedsec.com/2018/05/art_of_kerberoast/)
**Log Source**: Windows Security Event Log (Domain Controllers)
**Event ID**: 4769
**Status**: 0x0 (Audit Success)
**Ticket Encryption**: 0x17 (RC4)
**Service Name**: NOT krbtgt or NOT a system account (account name ends in $)
**TargetUserName**: NOT a system account (*$@*)


Monitor and alert on unusual events from an unexpected IP.
Note: There are potential false positives so whitelist normal source IPs and
manage risk of insecure ticket generation.


```yaml
name: Windows.Events.Kerberoasting
description: |
  **Description**:
  This Artifact will monitor all successful Kerberos TGS Ticket events for
  Service Accounts (SPN attribute) implemented with weak encryption. These
  tickets are vulnerable to brute force attack and this event is an indicator
  of a Kerberoasting attack.

  **ATT&CK**: [T1208 - Kerberoasting](https://attack.mitre.org/techniques/T1208/)
  Typical attacker methodology is to firstly request accounts in the domain
  with SPN attributes, then request an insecure TGS ticket for brute forcing.
  This attack is particularly effective as any domain credentials can be used
  to implement the attack and service accounts often have elevated privileges.
  Kerberoasting can be used for privilege escalation or persistence by adding a
  SPN attribute to an unexpected account.

  **Reference**: [The Art of Detecting Kerberoast Attacks](https://www.trustedsec.com/2018/05/art_of_kerberoast/)
  **Log Source**: Windows Security Event Log (Domain Controllers)
  **Event ID**: 4769
  **Status**: 0x0 (Audit Success)
  **Ticket Encryption**: 0x17 (RC4)
  **Service Name**: NOT krbtgt or NOT a system account (account name ends in $)
  **TargetUserName**: NOT a system account (*$@*)


  Monitor and alert on unusual events from an unexpected IP.
  Note: There are potential false positives so whitelist normal source IPs and
  manage risk of insecure ticket generation.


author: Matt Green - @mgreen27

type: CLIENT_EVENT

parameters:
  - name: eventLog
    default: C:\Windows\system32\winevt\logs\Security.evtx

sources:
  - name: Kerberoasting
    query: |
      LET files = SELECT * FROM glob(globs=eventLog)

      SELECT timestamp(epoch=System.TimeCreated.SystemTime) As EventTime,
              System.EventID.Value as EventID,
              System.Computer as Computer,
              EventData.ServiceName as ServiceName,
              EventData.ServiceSid as ServiceSid,
              EventData.TargetUserName as TargetUserName,
              "0x" + format(format="%x", args=EventData.Status) as Status,
              EventData.TargetDomainName as TargetDomainName,
              "0x" + format(format="%x", args=EventData.TicketEncryptionType) as TicketEncryptionType,
              "0x" + format(format="%x", args=EventData.TicketOptions) as TicketOptions,
              EventData.TransmittedServices as TransmittedServices,
              EventData.IpAddress as IpAddress,
              EventData.IpPort as IpPort
        FROM foreach(
          row=files,
          async=TRUE,
          query={
            SELECT *
            FROM watch_evtx(filename=OSPath)
            WHERE System.EventID.Value = 4769
                AND EventData.TicketEncryptionType = 23
                AND EventData.Status = 0
                AND NOT EventData.ServiceName =~ "krbtgt|\\$$"
                AND NOT EventData.TargetUserName =~ "\\$@"
        })

```
