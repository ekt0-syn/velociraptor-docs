---
title: Windows.Forensics.SRUM
hidden: true
tags: [Client Artifact]
---

Process the SRUM database.


```yaml
name: Windows.Forensics.SRUM
description: |
  Process the SRUM database.

reference:
  - https://www.sans.org/cyber-security-summit/archives/file/summit-archive-1492184583.pdf
  - https://cyberforensicator.com/2017/08/06/windows-srum-forensics/

type: client

parameters:
  - name: SRUMLocation
    default: c:/windows/system32/sru/srudb.dat
  - name: accessor
    default: auto
  - name: ExecutableRegex
    default: .
  - name: NetworkConnectionsGUID
    default: "{DD6636C4-8929-4683-974E-22C046A43763}"
    type: hidden
  - name: ApplicationResourceUsageGUID
    default: "{D10CA2FE-6FCF-4F6D-848E-B2E99266FA89}"
    type: hidden
  - name: ExecutionGUID
    default: "{5C8CF1C7-7257-4F13-B223-970EF5939312}"
    type: hidden
  - name: NetworkUsageGUID
    default: "{973F5D5C-1D90-4944-BE8E-24B94231A174}"
    type: hidden
  - name: Upload
    description: Select to Upload the SRUM database file 'srudb.dat'
    type: bool

export: |
  LET resolveESEId(OSPath, Accessor, Id) = cache(
      name="ESE",
      func=srum_lookup_id(file=OSPath, accessor=Accessor, id=Id),
      key=format(format="%v-%v-%v", args=[OSPath, Accessor, Id]))

  LET lookupSIDCache(OSPath, Accessor, Id) = cache(
      name="SID",
      func=lookupSID(sid=srum_lookup_id(file=OSPath, accessor=Accessor, id=Id)),
      key=format(format="%v-%v-%v", args=[OSPath, Accessor, Id]))

sources:
  - name: Upload
    precondition:
        SELECT * FROM scope() WHERE Upload
    query: |
        SELECT upload(file=SRUMLocation, accessor=accessor) AS Upload
        FROM scope()

  - name: Execution Stats
    query: |
        LET SRUMFiles <= SELECT OSPath FROM glob(globs=SRUMLocation)

        SELECT  AutoIncId AS ID,
                TimeStamp,
                resolveESEId(OSPath=SRUMFiles.OSPath,
                             Accessor=accessor, Id=AppId) AS App,
                resolveESEId(OSPath=SRUMFiles.OSPath,
                             Accessor=accessor, Id=UserId) AS UserSid,
                lookupSIDCache(OSPath=SRUMFiles.OSPath,
                          Accessor=accessor, Id=UserId) AS User,
                timestamp(winfiletime=EndTime) AS EndTime,
                DurationMS,
                NetworkBytesRaw
        FROM parse_ese(file=SRUMFiles.OSPath,
                       accessor=accessor, table=ExecutionGUID)
        WHERE App =~ ExecutableRegex

  - name: Application Resource Usage
    query: |
        LET SRUMFiles <= SELECT OSPath FROM glob(globs=SRUMLocation)

        SELECT AutoIncId as SRUMId,
               TimeStamp,
               resolveESEId(OSPath=SRUMFiles.OSPath,
                            Accessor=accessor, Id=AppId) AS App,
               resolveESEId(OSPath=SRUMFiles.OSPath,
                            Accessor=accessor, Id=UserId) AS UserSid,
               lookupSIDCache(OSPath=SRUMFiles.OSPath,
                         Accessor=accessor, Id=UserId) AS User,
               ForegroundCycleTime,
               BackgroundCycleTime,
               FaceTime,
               ForegroundContextSwitches,
               BackgroundContextSwitches,
               ForegroundBytesRead,
               ForegroundBytesWritten,
               ForegroundNumReadOperations,
               ForegroundNumWriteOperations,
               ForegroundNumberOfFlushes,
               BackgroundBytesRead,
               BackgroundBytesWritten,
               BackgroundNumReadOperations,
               BackgroundNumWriteOperations,
               BackgroundNumberOfFlushes
        FROM parse_ese(file=SRUMFiles.OSPath,
                       accessor=accessor, table=ApplicationResourceUsageGUID)
        WHERE App =~ ExecutableRegex

  - name: Network Connections
    query: |
        LET SRUMFiles <= SELECT OSPath FROM glob(globs=SRUMLocation)

        SELECT AutoIncId as SRUMId,
             TimeStamp,
             resolveESEId(OSPath=SRUMFiles.OSPath,
                          Accessor=accessor, Id=AppId) AS App,
             resolveESEId(OSPath=SRUMFiles.OSPath,
                          Accessor=accessor, Id=UserId) AS UserSid,
             lookupSIDCache(OSPath=SRUMFiles.OSPath,
                       Accessor=accessor, Id=UserId) AS User,
             InterfaceLuid,
             ConnectedTime,
             timestamp(winfiletime=ConnectStartTime) AS StartTime
        FROM parse_ese(file=SRUMFiles.OSPath,
                       accessor=accessor, table=NetworkConnectionsGUID)
        WHERE App =~ ExecutableRegex

  - name: Network Usage
    query: |
        LET SRUMFiles <= SELECT OSPath FROM glob(globs=SRUMLocation)

        SELECT AutoIncId as SRUMId,
             TimeStamp,
             resolveESEId(OSPath=SRUMFiles.OSPath,
                          Accessor=accessor, Id=AppId) AS App,
             resolveESEId(OSPath=SRUMFiles.OSPath,
                          Accessor=accessor, Id=UserId) AS UserSid,
             lookupSID(OSPath=SRUMFiles.OSPath,
                       Accessor=accessor, Id=UserId) AS User,
             UserId,
             BytesSent,
             BytesRecvd,
             InterfaceLuid,
             L2ProfileId,
             L2ProfileFlags
        FROM parse_ese(file=SRUMFiles.OSPath,
                       accessor=accessor, table=NetworkUsageGUID)
        WHERE App =~ ExecutableRegex

    notebook:
        - type: vql_suggestion
          name: SRUM Network Usage summary
          template: |
              /*
              # SRUM Network Usage summary
              */
              SELECT Fqdn,
                  count() as TotalEntries,
                  min(item=TimeStamp) as Earliest,
                  max(item=TimeStamp) as Latest,
                  App,
                  User,UserId,
                  sum(item=BytesSent) as TotalSent,
                  sum(item=BytesRecvd) as TotalRecvd,
                  InterfaceLuid
              FROM source(artifact="Windows.Forensics.SRUM/Network Usage")
              GROUP BY App, User,InterfaceLuid
              ORDER BY TotalSent DESC

```
