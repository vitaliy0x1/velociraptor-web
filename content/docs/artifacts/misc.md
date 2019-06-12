---
description: Various Artifacts which do not fit into other categories.
linktitle: Miscelaneous
menu:
  docs: {parent: Artifacts, weight: 30}
title: Miscelaneous Artifacts
toc: true

---
## Admin.Client.Upgrade

Remotely push new client updates.

NOTE: The updates can be pulled from any web server. You need to
ensure they are properly secured with SSL and at least a random
nonce in their path. You may configure the Velociraptor server to
serve these through the public directory.


Arg|Default|Description
---|------|-----------
clientURL|http://127.0.0.1:8000/public/velociraptor.exe|
configURL|http://127.0.0.1:8000/public/client.config.yaml|

{{% expand  "View Artifact Source" %}}


```
name: Admin.Client.Upgrade
description: |
  Remotely push new client updates.

  NOTE: The updates can be pulled from any web server. You need to
  ensure they are properly secured with SSL and at least a random
  nonce in their path. You may configure the Velociraptor server to
  serve these through the public directory.

parameters:
  - name: clientURL
    default: http://127.0.0.1:8000/public/velociraptor.exe
  - name: configURL
    default: http://127.0.0.1:8000/public/client.config.yaml

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'
    queries:
      - |
        /* This query fetches the binary and config and stores them in
         temp files. Note that tempfiles will be automatically
         cleaned at query end.
         */
        LET tmpfiles <= SELECT tempfile(
           data=query(vql={
             SELECT Content
             FROM http_client(url=clientURL, chunk_size=30000000)
           }),
           extension=".exe") as Binary,
        tempfile(
           data=query(vql={
             SELECT Content
             FROM http_client(url=configURL)
           })) as Config from scope()

      - |
        // Run the installer.
        SELECT * from foreach(
         row=tmpfiles,
         query={
            SELECT * from execve(
               argv=[Binary, "--config", Config, "-v", "service", "install"]
            )
         })
```
   {{% /expand %}}

## Admin.Events.PostProcessUploads

Sometimes we would like to post process uploads collected as part of
the hunt's artifact collections

Post processing means to watch the hunt for completed flows and run
a post processing command on the files obtained from each host.

The command will receive the list of paths of the files uploaded by
the artifact. We dont actually care what the command does with those
files - we will just relay our stdout/stderr to the artifact's
result set.


Arg|Default|Description
---|------|-----------
uploadPostProcessCommand|["/bin/ls", "-l"]\n|The command to run - must be a json array of strings! The list\nof files will be appended to the end of the command.\n
uploadPostProcessArtifact|Windows.Registry.NTUser.Upload|The name of the artifact to watch.\n

{{% expand  "View Artifact Source" %}}


```
name: Admin.Events.PostProcessUploads
description: |
  Sometimes we would like to post process uploads collected as part of
  the hunt's artifact collections

  Post processing means to watch the hunt for completed flows and run
  a post processing command on the files obtained from each host.

  The command will receive the list of paths of the files uploaded by
  the artifact. We dont actually care what the command does with those
  files - we will just relay our stdout/stderr to the artifact's
  result set.

type: SERVER_EVENT

parameters:
  - name: uploadPostProcessCommand
    description: |
      The command to run - must be a json array of strings! The list
      of files will be appended to the end of the command.
    default: |
      ["/bin/ls", "-l"]

  - name: uploadPostProcessArtifact
    description: |
      The name of the artifact to watch.
    default: Windows.Registry.NTUser.Upload

sources:
  - precondition:
      SELECT server_config FROM scope()
    queries:
      - |
        LET files = SELECT Flow,
            array(a1=parse_json_array(data=uploadPostProcessCommand),
                  a2=file_store(path=Flow.FlowContext.uploaded_files)) as Argv
        FROM watch_monitoring(artifact='System.Flow.Completion')
        WHERE uploadPostProcessArtifact in Flow.FlowContext.artifacts

      - |
        SELECT * from foreach(
          row=files,
          query={
             SELECT Flow.Urn as FlowUrn, Argv,
                    Stdout, Stderr, ReturnCode
             FROM execve(argv=Argv)
          })
```
   {{% /expand %}}

## Admin.System.CompressUploads

Compresses all uploaded files.

When artifacts collect files they are normally stored on the server
uncompressed. This artifact watches all completed flows and
compresses the files in the file store when the flow completes. This
is very useful for cloud based deployments with limited storage
space or when collecting large files.

In order to run this artifact you would normally run it as part of
an artifact acquisition process:

```
$ velociraptor --config /etc/server.config.yaml artifacts acquire Admin.System.CompressUploads
```

Note that there is nothing special about compressed files - you can
also just run `find` and `gzip` in the file store. Velociraptor will
automatically decompress the file when displaying it in the GUI
text/hexdump etc.


Arg|Default|Description
---|------|-----------
blacklistCompressionFilename|(?i).+ntuser.dat|Filenames which match this regex will be excluded from compression.

{{% expand  "View Artifact Source" %}}


```
name: Admin.System.CompressUploads
description: |
  Compresses all uploaded files.

  When artifacts collect files they are normally stored on the server
  uncompressed. This artifact watches all completed flows and
  compresses the files in the file store when the flow completes. This
  is very useful for cloud based deployments with limited storage
  space or when collecting large files.

  In order to run this artifact you would normally run it as part of
  an artifact acquisition process:

  ```
  $ velociraptor --config /etc/server.config.yaml artifacts acquire Admin.System.CompressUploads
  ```

  Note that there is nothing special about compressed files - you can
  also just run `find` and `gzip` in the file store. Velociraptor will
  automatically decompress the file when displaying it in the GUI
  text/hexdump etc.

type: SERVER_EVENT

parameters:
  - name: blacklistCompressionFilename
    description: Filenames which match this regex will be excluded from compression.
    default: '(?i).+ntuser.dat'

sources:
  - precondition:
      SELECT server_config FROM scope()
    queries:
      - |
        LET files = SELECT ClientId,
            Flow.Urn as Flow,
            Flow.FlowContext.uploaded_files as Files
        FROM watch_monitoring(artifact='System.Flow.Completion')
        WHERE Files and not Files =~ blacklistCompressionFilename

      - |
        SELECT ClientId, Flow, Files,
               compress(path=Files) as CompressedFiles
        FROM files
```
   {{% /expand %}}

## Demo.Plugins.Fifo

This is a demo of the fifo() plugin. The Fifo plugin collects and
caches rows from its inner query. Every subsequent execution of the
query then reads from the cache. The plugin will expire old rows
depending on its expiration policy - so we always see recent rows.

You can use this to build queries which consider historical events
together with current events at the same time. In this example, we
check for a successful logon preceeded by a number of failed logon
attempts.

In this example, we use the clock() plugin to simulate events. We
simulate failed logon attempts using the clock() plugin every
second. By feeding the failed logon events to the fifo() plugin we
ensure the fifo() plugin cache contains the last 5 failed logon
events.

We simulate a successful logon event every 3 seconds, again using
the clock plugin. Once a successful logon event is detected, we go
back over the last 5 login events, count them and collect the last
failed logon times (using the GROUP BY operator we group the
FailedTime for every unique SuccessTime).

If we receive more than 3 events, we emit the row.

This now represents a high value signal! It will only occur when a
successful logon event is preceeded by at least 3 failed logon
events in the last hour. It is now possible to escalate this on the
server via email or other alerts.

Here is sample output:

.. code-block:: json

    {
      "Count": 5,
      "FailedTime": [
        1549527272,
        1549527273,
        1549527274,
        1549527275,
        1549527276
      ],
      "SuccessTime": 1549527277
    }

Of course in the real artifact we would want to include more
information than just times (i.e. who logged on to where etc).


{{% expand  "View Artifact Source" %}}


```
name: Demo.Plugins.Fifo
description: |
  This is a demo of the fifo() plugin. The Fifo plugin collects and
  caches rows from its inner query. Every subsequent execution of the
  query then reads from the cache. The plugin will expire old rows
  depending on its expiration policy - so we always see recent rows.

  You can use this to build queries which consider historical events
  together with current events at the same time. In this example, we
  check for a successful logon preceeded by a number of failed logon
  attempts.

  In this example, we use the clock() plugin to simulate events. We
  simulate failed logon attempts using the clock() plugin every
  second. By feeding the failed logon events to the fifo() plugin we
  ensure the fifo() plugin cache contains the last 5 failed logon
  events.

  We simulate a successful logon event every 3 seconds, again using
  the clock plugin. Once a successful logon event is detected, we go
  back over the last 5 login events, count them and collect the last
  failed logon times (using the GROUP BY operator we group the
  FailedTime for every unique SuccessTime).

  If we receive more than 3 events, we emit the row.

  This now represents a high value signal! It will only occur when a
  successful logon event is preceeded by at least 3 failed logon
  events in the last hour. It is now possible to escalate this on the
  server via email or other alerts.

  Here is sample output:

  .. code-block:: json

      {
        "Count": 5,
        "FailedTime": [
          1549527272,
          1549527273,
          1549527274,
          1549527275,
          1549527276
        ],
        "SuccessTime": 1549527277
      }

  Of course in the real artifact we would want to include more
  information than just times (i.e. who logged on to where etc).
type: CLIENT_EVENT

sources:
  - queries:
      # This query simulates failed logon attempts.
      - |
        LET failed_logon = SELECT Unix as FailedTime from clock(period=1)

      # This is the fifo which holds the last 5 failed logon attempts
      # within the last hour.
      - |
        LET last_5_events = SELECT FailedTime
            FROM fifo(query=failed_logon, max_rows=5, max_age=3600)

      # We need to get it started collecting data immediately by
      # materializing the cache contents. Otherwise the fifo wont
      # start until it is first called (i.e. the first successful
      # login and we will miss the failed events before hand).
      - |
        LET foo <= SELECT * FROM last_5_events

      # This simulates successful logon - we assume every 3 seonds.
      - |
        LET success_logon = SELECT Unix as SuccessTime from clock(period=3)

      # For each successful logon, query the last failed logon
      # attempts from the fifo(). We also count the total number of
      # failed logons. We only actually emit results if there are more
      # than 3 failed logon attempts before each successful one.
      - |
        SELECT * FROM foreach(
          row=success_logon,
          query={
           SELECT SuccessTime,
              enumerate(items=FailedTime) as FailedTime,
              count(items=FailedTime) as Count
           FROM last_5_events GROUP BY SuccessTime
          }) WHERE Count > 3
```
   {{% /expand %}}

## Generic.Applications.Office.Keywords

Microsoft Office documents among other document format (such as
LibraOffice) are actually stored in zip files. The zip file contain
the document encoded as XML in a number of zip members.

This makes it difficult to search for keywords within office
documents because the ZIP files are typically compressed.

This artifact searches for office documents by file extension and
glob then uses the zip filesystem accessor to launch a yara scan
again the uncompressed data of the document. Keywords are more
likely to match when scanning the decompressed XML data.

The artifact returns a context around the keyword hit.

NOTE: The InternalMtime column shows the creation time of the zip
member within the document which may represent when the document was
initially created.

See
https://en.wikipedia.org/wiki/List_of_Microsoft_Office_filename_extensions
https://wiki.openoffice.org/wiki/Documentation/OOo3_User_Guides/Getting_Started/File_formats


Arg|Default|Description
---|------|-----------
documentGlobs|/*.{docx,docm,dotx,dotm,docb,xlsx,xlsm,xltx,xltm,pptx,pptm,potx,potm,ppam,ppsx,ppsm,sldx,sldm,odt,ott,oth,odm}|
searchGlob|C:\\Users\\**|
yaraRule|rule Hit {\n  strings:\n    $a = "secret" wide nocase\n    $b = "secret" nocase\n\n  condition:\n    any of them\n}\n|

{{% expand  "View Artifact Source" %}}


```
name: Generic.Applications.Office.Keywords
description: |
  Microsoft Office documents among other document format (such as
  LibraOffice) are actually stored in zip files. The zip file contain
  the document encoded as XML in a number of zip members.

  This makes it difficult to search for keywords within office
  documents because the ZIP files are typically compressed.

  This artifact searches for office documents by file extension and
  glob then uses the zip filesystem accessor to launch a yara scan
  again the uncompressed data of the document. Keywords are more
  likely to match when scanning the decompressed XML data.

  The artifact returns a context around the keyword hit.

  NOTE: The InternalMtime column shows the creation time of the zip
  member within the document which may represent when the document was
  initially created.

  See
  https://en.wikipedia.org/wiki/List_of_Microsoft_Office_filename_extensions
  https://wiki.openoffice.org/wiki/Documentation/OOo3_User_Guides/Getting_Started/File_formats

parameters:
  - name: documentGlobs
    default: /*.{docx,docm,dotx,dotm,docb,xlsx,xlsm,xltx,xltm,pptx,pptm,potx,potm,ppam,ppsx,ppsm,sldx,sldm,odt,ott,oth,odm}
  - name: searchGlob
    default: C:\Users\**
  - name: yaraRule
    default: |
      rule Hit {
        strings:
          $a = "secret" wide nocase
          $b = "secret" nocase

        condition:
          any of them
      }

sources:
  - queries:
      - |
        LET office_docs = SELECT FullPath AS OfficePath,
             timestamp(epoch=Mtime.Sec) as OfficeMtime,
             Size as OfficeSize
          FROM glob(globs=searchGlob + documentGlobs)

      # A list of zip members inside the doc that have some content.
      - |
        LET document_parts = SELECT OfficePath,
             FullPath AS ZipMemberPath
          FROM glob(globs=url(
             scheme="file", path=OfficePath, fragment="/**").String,
             accessor='zip')
          WHERE not IsDir and Size > 0

      # For each document, scan all its parts for the keyword.
      - |
        SELECT OfficePath,
               OfficeMtime,
               OfficeSize,
               File.ModTime as InternalMtime,
               Strings.HexData as HexContext
         FROM foreach(
           row=office_docs,
           query={
              SELECT File, Strings, OfficePath,
                     OfficeMtime, OfficeSize
              FROM yara(
                 rules=yaraRule,
                 files=document_parts.ZipMemberPath,
                 context=200,
                 accessor='zip')
         })
```
   {{% /expand %}}

## Generic.Client.Stats

An Event artifact which generates client's CPU and memory statistics.

Arg|Default|Description
---|------|-----------
Frequency|10|Return stats every this many seconds.

{{% expand  "View Artifact Source" %}}


```
name: Generic.Client.Stats
description: An Event artifact which generates client's CPU and memory statistics.
parameters:
  - name: Frequency
    description: Return stats every this many seconds.
    default: "10"
type: CLIENT_EVENT

sources:
  - queries:
      - |
        SELECT * from foreach(
         row={
           SELECT UnixNano FROM clock(period=atoi(string=Frequency))
         },
         query={
           SELECT UnixNano / 1000000000 as Timestamp,
                  Times.user + Times.system as CPU,
                  MemoryInfo.RSS as RSS
           FROM pslist(pid=getpid())
         })


reports:
  - type: SERVER_EVENT
    template: |
      {{ define "resources" }}
           SELECT Timestamp, rate(x=CPU, y=Timestamp) * 100 As CPUPercent,
                  RSS / 1000000 AS MemoryUse
           FROM source()
           WHERE CPUPercent >= 0
      {{ end }}

      {{ Query "resources" | LineChart "xaxis_mode" "time" "RSS.yaxis" 2 }}

  - type: MONITORING_DAILY
    template: |
      {{ define "resources" }}
           SELECT Timestamp, rate(x=CPU, y=Timestamp) * 100 As CPUPercent,
                  RSS / 1000000 AS MemoryUse
           FROM source()
           WHERE CPUPercent >= 0
      {{ end }}

      {{ $client_info := Query "SELECT * FROM clients(client_id=ClientId) LIMIT 1" }}

      # Client Footprint for {{ Get $client_info "0.OsInfo.Fqdn" }}

      The client has a client ID of {{ Get $client_info "0.ClientId" }}.
      Clients report the Velociraptor process footprint to the
      server every 10 seconds. The data includes the total CPU
      utilization, and the resident memory size used by the client.

      The following graph shows the total utilization. Memory
      utilization is meausred in `Mb` while CPU Utilization is
      measured by `Percent of one core`.

      We would expect the client to use around 1-5% of one core when
      idle, but if a heavy hunt is running this might climb
      substantially.

      {{ Query "resources" | LineChart "xaxis_mode" "time" "RSS.yaxis" 2 }}

      ## VQL Query

      The following VQL query was used to plot the graph above.

      ```sql
      {{ template "resources" }}
      ```

      > To learn about managing end point performance with Velociraptor see
        the [blog post](https://docs.velociraptor.velocidex.com/blog/html/2019/02/10/velociraptor_performance.html).
```
   {{% /expand %}}

## Generic.Forensic.Carving.URLs

Carve URLs from files located in a glob. Note that we do not parse
any files - we simply carve anything that looks like a URL.


Arg|Default|Description
---|------|-----------
UrlGlob|["C:/Documents and Settings/*/Local Settings/Application Data/Google/Chrome/User Data/**",\n "C:/Users/*/AppData/Local/Google/Chrome/User Data/**",\n "C:/Documents and Settings/*/Local Settings/History/**",\n "C:/Documents and Settings/*/Local Settings/Temporary Internet Files/**",\n "C:/Users/*/AppData/Local/Microsoft/Windows/WebCache/**",\n "C:/Users/*/AppData/Local/Microsoft/Windows/INetCache/**",\n "C:/Users/*/AppData/Local/Microsoft/Windows/INetCookies/**",\n "C:/Users/*/AppData/Roaming/Mozilla/Firefox/Profiles/**",\n "C:/Documents and Settings/*/Application Data/Mozilla/Firefox/Profiles/**"\n ]\n|

{{% expand  "View Artifact Source" %}}


```
name: Generic.Forensic.Carving.URLs
description: |
  Carve URLs from files located in a glob. Note that we do not parse
  any files - we simply carve anything that looks like a URL.


parameters:
  - name: UrlGlob
    default: |
      ["C:/Documents and Settings/*/Local Settings/Application Data/Google/Chrome/User Data/**",
       "C:/Users/*/AppData/Local/Google/Chrome/User Data/**",
       "C:/Documents and Settings/*/Local Settings/History/**",
       "C:/Documents and Settings/*/Local Settings/Temporary Internet Files/**",
       "C:/Users/*/AppData/Local/Microsoft/Windows/WebCache/**",
       "C:/Users/*/AppData/Local/Microsoft/Windows/INetCache/**",
       "C:/Users/*/AppData/Local/Microsoft/Windows/INetCookies/**",
       "C:/Users/*/AppData/Roaming/Mozilla/Firefox/Profiles/**",
       "C:/Documents and Settings/*/Application Data/Mozilla/Firefox/Profiles/**"
       ]

sources:
  - queries:
      - |
        LET matching = SELECT FullPath FROM glob(
            globs=parse_json_array(data=UrlGlob))
      - |
        SELECT FullPath, URL FROM foreach(
          row=matching,
          query={
            SELECT FullPath,
                   URL FROM parse_records_with_regex(file=FullPath,
               regex="(?P<URL>https?:\\/\\/[\\w\\.-]+[\\/\\w \\.-]*)")
          })
```
   {{% /expand %}}

## Generic.Forensic.Timeline

This artifact generates a timeline of a file glob in bodyfile
format. We currently do not calculate the md5 because it is quite
expensive.


Arg|Default|Description
---|------|-----------
timelineGlob|C:\\Users\\**|
timelineAccessor|file|

{{% expand  "View Artifact Source" %}}


```
name: Generic.Forensic.Timeline
description: |
  This artifact generates a timeline of a file glob in bodyfile
  format. We currently do not calculate the md5 because it is quite
  expensive.

parameters:
  - name: timelineGlob
    default: C:\Users\**
  - name: timelineAccessor
    default: file

sources:
  # For NTFS accessors we write the MFT id as the inode. On windows
  # the file accessor does not give the inode at all.
  - precondition:
      SELECT OS From info() where OS = 'windows' AND timelineAccessor = 'ntfs'
    queries:
      - |
        SELECT 0 AS Md5, FullPath,
               Sys.mft as Inode,
               Mode.String AS Mode, 0 as Uid, 0 as Gid, Size,
               Atime.Sec AS Atime, Mtime.Sec AS Mtime, Ctime.Sec AS Ctime
        FROM glob(globs=timelineGlob, accessor=timelineAccessor)

  # For linux we can get the Inode from Sys.Ino
  - precondition:
      SELECT * From scope() where timelineAccessor = 'file'
    queries:
      - |
        SELECT 0 AS Md5, FullPath,
               Sys.Ino as Inode,
               Mode.String AS Mode, Sys.Uid AS Uid, Sys.Gid AS Gid, Size,
               Atime.Sec AS Atime, Mtime.Sec AS Mtime, Ctime.Sec AS Ctime
        FROM glob(globs=timelineGlob, accessor=timelineAccessor)
```
   {{% /expand %}}

## Network.ExternalIpAddress

Detect the external ip address of the end point.

Arg|Default|Description
---|------|-----------
externalUrl|http://www.myexternalip.com/raw|The URL of the external IP detection site.

{{% expand  "View Artifact Source" %}}


```
name: Network.ExternalIpAddress
description: Detect the external ip address of the end point.
parameters:
  - name: externalUrl
    default: http://www.myexternalip.com/raw
    description: The URL of the external IP detection site.
sources:
  - precondition: SELECT * from info()
    queries:
      - |
        SELECT Content as IP from http_client(url=externalUrl)
```
   {{% /expand %}}

## Reporting.Hunts.Details

Report details about which client ran each hunt, how long it took
and if it has completed.


{{% expand  "View Artifact Source" %}}


```
name: Reporting.Hunts.Details
description: |
  Report details about which client ran each hunt, how long it took
  and if it has completed.

sources:
  - precondition:
      SELECT server_config FROM scope()

    queries:
      - |
        LET hunts = SELECT basename(path=hunt_id) as hunt_id,
            create_time,
            hunt_description
        FROM hunts() order by create_time desc limit 6
      - |
        LET flows = select hunt_id,
          hunt_description,
          Fqdn,
          ClientId,
          { SELECT os_info.system FROM clients(search=ClientId) } as OS,
          timestamp(epoch=Flow.FlowContext.create_time/1000000) as create_time,
          basename(path=Flow.Urn) as flow_id,
          (Flow.FlowContext.active_time - Flow.FlowContext.create_time) / 1000000 as Duration,
          format(format='%v', args=[Flow.FlowContext.state]) as State
        FROM hunt_flows(hunt_id=hunt_id) order by create_time desc
      - |
        SELECT * from foreach(row=hunts, query=flows)
```
   {{% /expand %}}

## System.Hunt.Participation

Endpoints may participate in hunts. This artifact collects which hunt
each system participated in.

Note: This is an automated system hunt. You do not need to start it.


{{% expand  "View Artifact Source" %}}


```
name: System.Hunt.Participation
description: |
     Endpoints may participate in hunts. This artifact collects which hunt
     each system participated in.

     Note: This is an automated system hunt. You do not need to start it.

reports:
  - type: MONITORING_DAILY
    template: |
      {{ define "all_hunts" }}LET allhunts <= SELECT * FROM hunts(){{ end }}
      {{ define "hunts" }}
           SELECT * FROM foreach(
             row={ SELECT timestamp(epoch=Timestamp) AS Scheduled,
                          HuntId as ParticipatedHuntId
                   FROM source(client_id=ClientId,
                       artifact='System.Hunt.Participation') },
             query={
                SELECT Scheduled,
                       HuntId,
                       HuntDescription,
                       StartRequest.Args.Artifacts.Names
                FROM allhunts
                WHERE HuntId = ParticipatedHuntId
             })
      {{ end }}

      {{ $client_info := Query "SELECT * FROM clients(client_id=ClientId) LIMIT 1" }}

      # Hunt participation for {{ Get $client_info "0.OsInfo.Fqdn" }}

      The client with a client ID of {{ Get $client_info "0.ClientId" }} participated in some hunts today.

      {{ Query "all_hunts" "hunts" | Table }}

      ## VQL Query
      The following VQL query was used to plot the graph above.

      ```sql
      {{ template "hunts" }}
      ```
```
   {{% /expand %}}

## Windows.Applications.ChocolateyPackages

Chocolatey packages installed in a system.

Arg|Default|Description
---|------|-----------
ChocolateyInstall||

{{% expand  "View Artifact Source" %}}


```
name: Windows.Applications.ChocolateyPackages
description: Chocolatey packages installed in a system.
parameters:
  - name: ChocolateyInstall
    default: ""

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'
    queries:
      - |
        LET files =
          SELECT FullPath, parse_xml(file=FullPath) AS Metadata
          -- Use the ChocolateyInstall parameter if it is set.
          FROM glob(globs=if(
             condition=ChocolateyInstall,
             then=ChocolateyInstall,
             -- Otherwise just use the environment.
             else=environ(var='ChocolateyInstall')) + '/lib/*/*.nuspec')
      - |
        SELECT * FROM if(
        condition={
            SELECT * FROM if(
               condition=ChocolateyInstall,
               then=ChocolateyInstall,
               else=environ(var="ChocolateyInstall"))
          },
        then={
            SELECT FullPath,
                   Metadata.package.metadata.id as Name,
                   Metadata.package.metadata.version as Version,
                   Metadata.package.metadata.summary as Summary,
                   Metadata.package.metadata.authors as Authors,
                   Metadata.package.metadata.licenseUrl as License
            FROM files
        })
```
   {{% /expand %}}

## Windows.Applications.Chrome.Extensions

Fetch Chrome extensions.

Chrome extensions are installed into the user's home directory.  We
search for manifest.json files in a known path within each system
user's home directory. We then parse the manifest file as JSON.

Many extensions use locale packs to resolve strings like name and
description. In this case we detect the default locale and load
those locale files. We then resolve the extension's name and
description from there.


Arg|Default|Description
---|------|-----------
extensionGlobs|\\AppData\\Local\\Google\\Chrome\\User Data\\*\\Extensions\\*\\*\\manifest.json|

{{% expand  "View Artifact Source" %}}


```
name: Windows.Applications.Chrome.Extensions
description: |
  Fetch Chrome extensions.

  Chrome extensions are installed into the user's home directory.  We
  search for manifest.json files in a known path within each system
  user's home directory. We then parse the manifest file as JSON.

  Many extensions use locale packs to resolve strings like name and
  description. In this case we detect the default locale and load
  those locale files. We then resolve the extension's name and
  description from there.

parameters:
  - name: extensionGlobs
    default: \AppData\Local\Google\Chrome\User Data\*\Extensions\*\*\manifest.json
sources:
  - precondition: |
      SELECT OS From info() where OS = 'windows'
    queries:
      - |
        /* For each user on the system, search for extension manifests
           in their home directory. */
        LET extension_manifests = SELECT * from foreach(
          row={
             SELECT Uid, Name AS User, Directory from Artifact.Windows.Sys.Users()
          },
          query={
             SELECT FullPath, Mtime, Ctime, User, Uid from glob(
               globs=Directory + extensionGlobs)
          })

      - |
        /* If the Manifest declares a default_locale then we
           load and parse the messages file. In this case the
           messages are actually stored in the locale file
           instead of the main manifest.json file.
        */
        LET maybe_read_locale_file =
           SELECT * from if(
              condition={
                 select * from scope() where Manifest.default_locale
              },
              then={
                 SELECT Manifest,
                        Uid, User,
                        Filename as LocaleFilename,
                        ManifestFilename,
                        parse_json(data=Data) AS LocaleManifest
                 FROM read_file(
                         -- Munge the filename to get the messages.json path.
                         filenames=regex_replace(
                           source=ManifestFilename,
                           replace="\\_locales\\" + Manifest.default_locale +
                                   "\\messages.json",
                           re="\\\\manifest.json$"))
              },
              else={
                  -- Just fill in empty Locale results.
                  SELECT Manifest,
                         Uid, User,
                         "" AS LocaleFilename,
                         "" AS ManifestFilename,
                         "" AS LocaleManifest
                  FROM scope()
              })

      - |
        LET parse_json_files = SELECT * from foreach(
           row={
             SELECT Filename as ManifestFilename,
                    Uid, User,
                    parse_json(data=Data) as Manifest
             FROM read_file(filenames=FullPath)
           },
           query=maybe_read_locale_file)

      - |
        LET parsed_manifest_files = SELECT * from foreach(
          row=extension_manifests,
          query=parse_json_files)

      - |
        SELECT Uid, User,

               /* If the manifest name contains __MSG_ then the real
                  name is stored in the locale manifest. This condition
                  resolves the Name column either to the main manifest or
                  the locale manifest.
               */
               if(condition="__MSG_" in Manifest.name,
                  then=get(item=LocaleManifest,
                     member=regex_replace(
                        source=Manifest.name,
                        replace="$1",
                        re="(?:__MSG_(.+)__)")).message,
                  else=Manifest.name) as Name,

               if(condition="__MSG_" in Manifest.description,
                  then=get(item=LocaleManifest,
                     member=regex_replace(
                        source=Manifest.description,
                        replace="$1",
                        re="(?:__MSG_(.+)__)")).message,
                  else=Manifest.description) as Description,

               /* Get the Identifier and Version from the manifest filename */
               regex_replace(
                 source=ManifestFilename,
                 replace="$1",
                 re="(?:.+Extensions\\\\([^\\\\]+)\\\\([^\\\\]+)\\\\manifest.json)$") AS Identifier,
               regex_replace(
                 source=ManifestFilename,
                 replace="$2",
                 re="(?:.+Extensions\\\\([^\\\\]+)\\\\([^\\\\]+)\\\\manifest.json)$") AS Version,

               Manifest.author as Author,
               Manifest.background.persistent AS Persistent,
               regex_replace(
                 source=ManifestFilename,
                 replace="$1",
                 re="(.+Extensions\\\\.+\\\\)manifest.json$") AS Path,

               Manifest.oauth2.scopes as Scopes,
               Manifest.permissions as Permissions,
               Manifest.key as Key

        FROM parsed_manifest_files
```
   {{% /expand %}}

## Windows.Applications.OfficeMacros

Office macros are a favourite initial infection vector. Many users
click through the warning dialogs.

This artifact scans through the given directory glob for common
office files. We then try to extract any embedded macros by parsing
the OLE file structure.

If a macro calls an external program (e.g. Powershell) this is very
suspicious!


Arg|Default|Description
---|------|-----------
officeExtensions|*.{xls,xlsm,doc,docx,ppt,pptm}|
officeFileSearchGlob|C:\\Users\\**\\|The directory to search for office documents.

{{% expand  "View Artifact Source" %}}


```
name: Windows.Applications.OfficeMacros
description: |
  Office macros are a favourite initial infection vector. Many users
  click through the warning dialogs.

  This artifact scans through the given directory glob for common
  office files. We then try to extract any embedded macros by parsing
  the OLE file structure.

  If a macro calls an external program (e.g. Powershell) this is very
  suspicious!

parameters:
  - name: officeExtensions
    default: "*.{xls,xlsm,doc,docx,ppt,pptm}"
  - name: officeFileSearchGlob
    default: C:\Users\**\
    description: The directory to search for office documents.

sources:
  - queries:
      - |
        SELECT * FROM foreach(
           row={
              SELECT FullPath FROM glob(globs=officeFileSearchGlob + officeExtensions)
           },
           query={
               SELECT * from olevba(file=FullPath)
           })
```
   {{% /expand %}}

## Windows.EventLogs.DHCP


This artifact parses the windows dhcp event log looking for evidence
of IP address assignments.

In some investigations it is important to be able to identify the
machine which was assigned a particular IP address at a point in
time. Usually these logs are available from the DHCP server, but in
many cases the server logs are not available (for example, if the
endpoint was visiting a different network or the DHCP server is on a
wireless router with no log retention).

On windows, there are two types of logs:

  1. The first type is the admin log
     (`Microsoft-Windows-Dhcp-Client%4Admin.evt`). These only contain
     errors such as an endpoint trying to continue its lease, but
     the lease is rejected by the server.

  2. The operational log
     (`Microsoft-Windows-Dhcp-Client%4Operational.evtx`) contains
     the full log of each lease. Unfortunately this log is disabled
     by default. If it is available we can rely on the information.


Arg|Default|Description
---|------|-----------
eventDirGlob|C:\\Windows\\system32\\winevt\\logs\\|
adminLog|Microsoft-Windows-Dhcp-Client%4Admin.evtx|
operationalLog|Microsoft-Windows-Dhcp-Client%4Operational.evtx|
accessor|file|

{{% expand  "View Artifact Source" %}}


```
name: Windows.EventLogs.DHCP
description: |

  This artifact parses the windows dhcp event log looking for evidence
  of IP address assignments.

  In some investigations it is important to be able to identify the
  machine which was assigned a particular IP address at a point in
  time. Usually these logs are available from the DHCP server, but in
  many cases the server logs are not available (for example, if the
  endpoint was visiting a different network or the DHCP server is on a
  wireless router with no log retention).

  On windows, there are two types of logs:

    1. The first type is the admin log
       (`Microsoft-Windows-Dhcp-Client%4Admin.evt`). These only contain
       errors such as an endpoint trying to continue its lease, but
       the lease is rejected by the server.

    2. The operational log
       (`Microsoft-Windows-Dhcp-Client%4Operational.evtx`) contains
       the full log of each lease. Unfortunately this log is disabled
       by default. If it is available we can rely on the information.

parameters:
  - name: eventDirGlob
    default: C:\Windows\system32\winevt\logs\
  - name: adminLog
    default: Microsoft-Windows-Dhcp-Client%4Admin.evtx

  - name: operationalLog
    default: Microsoft-Windows-Dhcp-Client%4Operational.evtx

  - name: accessor
    default: file

sources:
  - name: RejectedDHCP
    queries:
      - |
        LET files = SELECT * FROM glob(
            globs=eventDirGlob + adminLog,
            accessor=accessor)
      - |
        SELECT Time AS _Time,
               timestamp(epoch=Time) As Timestamp,
               Computer, MAC, ClientIP, DHCPServer, Type FROM foreach(
           row=files,
           query={
              SELECT System.TimeCreated.SystemTime as Time,
                     System.Computer AS Computer,
                     format(format="%x:%x:%x:%x:%x:%x", args=[EventData.HWAddress]) AS MAC,
                     ip(netaddr4_le=EventData.Address1) AS ClientIP,
                     ip(netaddr4_le=EventData.Address2) AS DHCPServer,
                     "Lease Rejected" AS Type
              FROM parse_evtx(filename=FullPath, accessor=accessor)
              WHERE System.EventID.Value = 1002
           })

  - name: AssignedDHCP
    queries:
      - |
        LET files = SELECT * FROM glob(
            globs=eventDirGlob + operationalLog,
            accessor=accessor)
      - |
        SELECT Time AS _Time,
               timestamp(epoch=Time) As Timestamp,
               Computer, MAC, ClientIP, DHCPServer, Type FROM foreach(
           row=files,
           query={
              SELECT System.TimeCreated.SystemTime as Time,
                     System.Computer AS Computer,
                     EventData.InterfaceGuid AS MAC,
                     ip(netaddr4_le=EventData.Address1) AS ClientIP,
                     ip(netaddr4_le=EventData.Address2) AS DHCPServer,
                     "Lease Assigned" AS Type
              FROM parse_evtx(filename=FullPath, accessor=accessor)
              WHERE System.EventID.Value = 60000
           })


reports:
  - type: CLIENT
    template: |
      Evidence of DHCP assigned IP addresses
      ======================================

      {{ .Description }}

      {{ define "assigned_dhcp" }}
            SELECT Computer, ClientIP,
                   count(items=Timestamp) AS Total,
                   enumerate(items=Timestamp) AS Times
            FROM source(source='AssignedDHCP')
            GROUP BY ClientIP
      {{ end }}
      {{ define "rejected_dhcp" }}
            SELECT Computer, ClientIP,
                   count(items=Timestamp) AS Total,
                   enumerate(items=Timestamp) AS Times
            FROM source(source='RejectedDHCP')
            GROUP BY ClientIP
      {{ end }}

      {{ $assigned := Query "assigned_dhcp"}}
      {{ if $assigned }}
      ## Operational logs

      This machine has DHCP operational logging enabled. We therefore
      can see complete references to all granted leases:
        {{ Table $assigned }}

      ## Timeline

      {{ Query "SELECT _Time * 1000, ClientIP FROM source(source='AssignedDHCP')" | Timeline }}

      {{ end }}

      ## Admin logs

      The admin logs show errors with DHCP lease requests. Typically
      rejected leases indicate that the machine held a least on a IP
      address in the past, but this lease is invalid for its current
      environment. For example, the machine has been moved to a
      different network.

      {{ Query "rejected_dhcp" | Table }}

      {{ Query "SELECT _Time * 1000, ClientIP FROM source(source='RejectedDHCP')" | Timeline }}
```
   {{% /expand %}}

## Windows.Network.ArpCache

Address resolution cache, both static and dynamic (from ARP, NDP).

Arg|Default|Description
---|------|-----------
wmiQuery|SELECT AddressFamily, Store, State, InterfaceIndex, IPAddress,\n       InterfaceAlias, LinkLayerAddress\nfrom MSFT_NetNeighbor\n|
wmiNamespace|ROOT\\StandardCimv2|
kMapOfState|{\n "0": "Unreachable",\n "1": "Incomplete",\n "2": "Probe",\n "3": "Delay",\n "4": "Stale",\n "5": "Reachable",\n "6": "Permanent",\n "7": "TBD"\n}\n|

{{% expand  "View Artifact Source" %}}


```
name: Windows.Network.ArpCache
description: Address resolution cache, both static and dynamic (from ARP, NDP).
parameters:
  - name: wmiQuery
    default: |
      SELECT AddressFamily, Store, State, InterfaceIndex, IPAddress,
             InterfaceAlias, LinkLayerAddress
      from MSFT_NetNeighbor
  - name: wmiNamespace
    default: ROOT\StandardCimv2

  - name: kMapOfState
    default: |
     {
      "0": "Unreachable",
      "1": "Incomplete",
      "2": "Probe",
      "3": "Delay",
      "4": "Stale",
      "5": "Reachable",
      "6": "Permanent",
      "7": "TBD"
     }

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'
    queries:
      - |
        LET interfaces <=
          SELECT Index, HardwareAddr, IP
          FROM Artifact.Windows.Network.InterfaceAddresses()

      - |
        LET arp_cache = SELECT if(condition=AddressFamily=23,
                    then="IPv6",
                  else=if(condition=AddressFamily=2,
                    then="IPv4",
                  else=AddressFamily)) as AddressFamily,

               if(condition=Store=0,
                    then="Persistent",
                  else=if(condition=(Store=1),
                    then="Active",
                  else="?")) as Store,

               get(item=parse_json(data=kMapOfState),
                   member=encode(string=State, type='string')) AS State,
               InterfaceIndex, IPAddress,
               InterfaceAlias, LinkLayerAddress
            FROM wmi(query=wmiQuery, namespace=wmiNamespace)
      - |
        SELECT * FROM foreach(
          row=arp_cache,
          query={
             SELECT AddressFamily, Store, State, InterfaceIndex,
                    IP AS LocalAddress, HardwareAddr, IPAddress as RemoteAddress,
                    InterfaceAlias, LinkLayerAddress AS RemoteMACAddress
             FROM interfaces
             WHERE InterfaceIndex = Index
          })
```
   {{% /expand %}}

## Windows.Network.InterfaceAddresses

Network interfaces and relevant metadata.

{{% expand  "View Artifact Source" %}}


```
name: Windows.Network.InterfaceAddresses
description: Network interfaces and relevant metadata.
sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'
    queries:
      - |
        LET interface_address =
           SELECT Index, MTU, Name, HardwareAddr, Flags, Addrs
           from interfaces()

      - |
        SELECT Index, MTU, Name, HardwareAddr.String As HardwareAddr,
           Flags, Addrs.IP as IP, Addrs.Mask.String as Mask
        FROM flatten(query=interface_address)
```
   {{% /expand %}}

## Windows.Network.ListeningPorts

Processes with listening (bound) network sockets/ports.

{{% expand  "View Artifact Source" %}}


```
name: Windows.Network.ListeningPorts
description: Processes with listening (bound) network sockets/ports.
sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'
    queries:
      - |
        LET process <= SELECT Name, Pid from pslist()

      - |
        SELECT * from foreach(
          row={
            SELECT Pid AS PortPid, Laddr.Port AS Port,
                   TypeString as Protocol, FamilyString as Family,
                   Laddr.IP as Address
            FROM netstat() where Status = 'LISTEN'
          },
          query={
            SELECT Pid, Name, Port, Protocol, Family, Address
            FROM process where Pid = PortPid
          })
```
   {{% /expand %}}

## Windows.Network.Netstat

Show information about open sockets. On windows the time when the
socket was first bound is also shown.


{{% expand  "View Artifact Source" %}}


```
name: Windows.Network.Netstat
description: |
  Show information about open sockets. On windows the time when the
  socket was first bound is also shown.

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'
    queries:
      - |
        SELECT Pid, FamilyString as Family,
               TypeString as Type,
               Status,
               Laddr.IP, Laddr.Port,
               Raddr.IP, Raddr.Port,
               Timestamp
               FROM netstat()
```
   {{% /expand %}}

## Windows.Packs.Autoexec

Aggregate of executables that will automatically execute on the
target machine. This is an amalgamation of other tables like
services, scheduled_tasks, startup_items and more.


{{% expand  "View Artifact Source" %}}


```
name: Windows.Packs.Autoexec
description: |
  Aggregate of executables that will automatically execute on the
  target machine. This is an amalgamation of other tables like
  services, scheduled_tasks, startup_items and more.

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'
    queries:
      - |
        SELECT * from chain(
          q1={
            SELECT Name, Command AS Path, "StartupItems" as Source
            FROM Artifact.Windows.Sys.StartupItems()
          })
```
   {{% /expand %}}

## Windows.Packs.Persistence

This artifact pack collects various persistence mechanisms in Windows.


{{% expand  "View Artifact Source" %}}


```
name: Windows.Packs.Persistence
description: |
  This artifact pack collects various persistence mechanisms in Windows.

precondition:
  SELECT OS from info() where OS = "windows"

sources:
  - name: WMI Event Filters
    description: |
      {{ DocFrom "Windows.Persistence.PermanentWMIEvents" }}

    queries:
      - |
        SELECT * FROM Artifact.Windows.Persistence.PermanentWMIEvents()

  - name: Startup Items
    description: |
      {{ DocFrom "Windows.Sys.StartupItems" }}

    queries:
      - |
        SELECT * FROM Artifact.Windows.Sys.StartupItems()

  - name: Debug Bootstraping
    description: |
      {{ DocFrom "Windows.Persistence.Debug" }}

      If there are any rows in the table below then executing the
      program will also launch the program listed under the Debugger
      column.

    queries:
      - SELECT * FROM Artifact.Windows.Persistence.Debug()
```
   {{% /expand %}}

## Windows.Registry.NTUser

This artifact searches for keys or values within the user's
NTUser.dat registry hives.

When a user logs into a windows machine the system creates their own
"profile" which consists of a registry hive mapped into the
HKEY_USERS hive. This hive file is locked as long as the user is
logged in. If the user is not logged in, the file is not mapped at
all.

This artifact bypasses the locking mechanism by parsing the raw NTFS
filesystem to recover the registry hives. We then parse the registry
hives to search for the glob provided.

This artifact is designed to be reused by other artifacts that need
to access user data.


Arg|Default|Description
---|------|-----------
KeyGlob|Software\\Microsoft\\Windows\\CurrentVersion\\Explorer\\ComDlg32\\**|
UserHomes|C:\\Users\\*\\NTUSER.DAT|

{{% expand  "View Artifact Source" %}}


```
name: Windows.Registry.NTUser
description: |
  This artifact searches for keys or values within the user's
  NTUser.dat registry hives.

  When a user logs into a windows machine the system creates their own
  "profile" which consists of a registry hive mapped into the
  HKEY_USERS hive. This hive file is locked as long as the user is
  logged in. If the user is not logged in, the file is not mapped at
  all.

  This artifact bypasses the locking mechanism by parsing the raw NTFS
  filesystem to recover the registry hives. We then parse the registry
  hives to search for the glob provided.

  This artifact is designed to be reused by other artifacts that need
  to access user data.

parameters:
 - name: KeyGlob
   default: Software\Microsoft\Windows\CurrentVersion\Explorer\ComDlg32\**
 - name: UserHomes
   default: C:\Users\*\NTUSER.DAT

sources:
 - queries:
     - |
       SELECT * FROM foreach(
         row={
            SELECT FullPath FROM glob(globs=UserHomes)
         },
         query={
            SELECT FullPath, Data, Mtime.Sec AS Mtime FROM glob(
               globs=url(scheme="ntfs",
                  path=FullPath,
                  fragment=KeyGlob).String,
               accessor="raw_reg")
         })
```
   {{% /expand %}}

## Windows.Registry.NTUser.Upload

This artifact collects all the user's NTUser.dat registry hives.

When a user logs into a windows machine the system creates their own
"profile" which consists of a registry hive mapped into the
HKEY_USERS hive. This hive file is locked as long as the user is
logged in.

This artifact bypasses the locking mechanism by extracting the
registry hives using raw NTFS parsing. We then just upload all hives
to the server.


{{% expand  "View Artifact Source" %}}


```
name: Windows.Registry.NTUser.Upload
description: |
  This artifact collects all the user's NTUser.dat registry hives.

  When a user logs into a windows machine the system creates their own
  "profile" which consists of a registry hive mapped into the
  HKEY_USERS hive. This hive file is locked as long as the user is
  logged in.

  This artifact bypasses the locking mechanism by extracting the
  registry hives using raw NTFS parsing. We then just upload all hives
  to the server.

sources:
  - precondition: |
      SELECT OS From info() where OS = 'windows'
    queries:
      - |
        LET users = SELECT Name, Directory as HomeDir
            FROM Artifact.Windows.Sys.Users()
            WHERE Directory

      - |
        SELECT upload(file="\\\\.\\" + HomeDir + "\\ntuser.dat",
                      accessor="ntfs") as Upload
        FROM users
```
   {{% /expand %}}

## Windows.Registry.Sysinternals.Eulacheck

Checks for the Accepted Sysinternals EULA from the registry key
"HKCU\Software\Sysinternals\[TOOL]\".  When a Sysinternals tool is
first run on a system, the EULA must be accepted. This writes a
value called EulaAccepted under that key.

Note: This artifact uses HKEY_USERS and therefore will not detect
users that are not currently logged on.


Arg|Default|Description
---|------|-----------
Sysinternals_Reg_Key|HKEY_USERS\\*\\Software\\Sysinternals\\*|

{{% expand  "View Artifact Source" %}}


```
name: Windows.Registry.Sysinternals.Eulacheck
description: |
  Checks for the Accepted Sysinternals EULA from the registry key
  "HKCU\Software\Sysinternals\[TOOL]\".  When a Sysinternals tool is
  first run on a system, the EULA must be accepted. This writes a
  value called EulaAccepted under that key.

  Note: This artifact uses HKEY_USERS and therefore will not detect
  users that are not currently logged on.

parameters:
   - name: Sysinternals_Reg_Key
     default: HKEY_USERS\*\Software\Sysinternals\*

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'

    queries:
    - |
      LET users <= SELECT Name, UUID FROM Artifact.Windows.Sys.Users()
    - |
      SELECT Key.Name as ProgramName,
             Key.FullPath as Key,
             timestamp(epoch=Key.Mtime.Sec) AS TimeAccepted,
             {
                SELECT Name FROM users WHERE UUID=regex_replace(
                   source=Key.FullPath, re=".+\\\\(S-[^\\\\]+)\\\\.+", replace="$1")
             } as User,
             EulaAccepted
      FROM read_reg_key(globs=split(string=Sysinternals_Reg_Key, sep=',[\\s]*'))
```
   {{% /expand %}}

## Windows.Search.FileFinder

Find files on the filesystem using the filename or content.


## Performance Note

This artifact can be quite expensive, especially if we search file
content. It will require opening each file and reading its entire
content. To minimize the impact on the endpoint we recommend this
artifact is collected with a rate limited way (about 20-50 ops per
second).

This artifact is usefull in the following scenarios:

  * We need to locate all the places on our network where customer
    data has been copied.

  * We’ve identified malware in a data breach, named using short
    random strings in specific folders and need to search for other
    instances across the network.

  * We believe our user account credentials have been dumped and
    need to locate them.

  * We need to search for exposed credit card data to satisfy PCI
    requirements.

  * We have a sample of data that has been disclosed and need to
    locate other similar files


Arg|Default|Description
---|------|-----------
SearchFilesGlob|C:\\Users\\**|Use a glob to define the files that will be searched.
Keywords|None|A comma delimited list of strings to search for.
Use_Raw_NTFS|Y|
Upload_File|N|
Calculate_Hash|N|
MoreRecentThan||
ModifiedBefore||

{{% expand  "View Artifact Source" %}}


```
name: Windows.Search.FileFinder
description: |
  Find files on the filesystem using the filename or content.


  ## Performance Note

  This artifact can be quite expensive, especially if we search file
  content. It will require opening each file and reading its entire
  content. To minimize the impact on the endpoint we recommend this
  artifact is collected with a rate limited way (about 20-50 ops per
  second).

  This artifact is usefull in the following scenarios:

    * We need to locate all the places on our network where customer
      data has been copied.

    * We’ve identified malware in a data breach, named using short
      random strings in specific folders and need to search for other
      instances across the network.

    * We believe our user account credentials have been dumped and
      need to locate them.

    * We need to search for exposed credit card data to satisfy PCI
      requirements.

    * We have a sample of data that has been disclosed and need to
      locate other similar files


precondition:
  SELECT * FROM info() where OS = 'windows'

parameters:
  - name: SearchFilesGlob
    default: C:\Users\**
    description: Use a glob to define the files that will be searched.

  - name: Keywords
    default:
    description: A comma delimited list of strings to search for.

  - name: Use_Raw_NTFS
    default: Y
    type: bool

  - name: Upload_File
    default: N
    type: bool

  - name: Calculate_Hash
    default: N
    type: bool

  - name: MoreRecentThan
    default: ""
    type: timestamp

  - name: ModifiedBefore
    default: ""
    type: timestamp


sources:
  - queries:
    - |
      LET ntfs_search = SELECT FullPath,
               Sys.mft as Inode,
               Mode.String AS Mode, Size,
               Mtime.Sec AS Modified,
               timestamp(epoch=Atime.Sec) AS ATime,
               timestamp(epoch=Mtime.Sec) AS MTime,
               timestamp(epoch=Ctime.Sec) AS CTime,
               if(condition=(Upload_File = "Y" and NOT IsDir ),
                  then=upload(file=FullPath, accessor="ntfs")) AS Upload,
               if(condition=(Calculate_Hash = "Y" and NOT IsDir ),
                  then=hash(path=FullPath, accessor="ntfs")) AS Hash
        FROM glob(globs=SearchFilesGlob, accessor="ntfs")

    - |
      LET file_search = SELECT FullPath,
               Sys.mft as Inode,
               Mode.String AS Mode, Size,
               Mtime.Sec AS Modified,
               timestamp(epoch=Atime.Sec) AS ATime,
               timestamp(epoch=Mtime.Sec) AS MTime,
               timestamp(epoch=Ctime.Sec) AS CTime,
               if(condition=(Upload_File = "Y" and NOT IsDir ),
                  then=upload(file=FullPath, accessor="file")) AS Upload,
               if(condition=(Calculate_Hash = "Y" and NOT IsDir ),
                  then=hash(path=FullPath, accessor="file")) AS Hash
        FROM glob(globs=SearchFilesGlob, accessor="file")

    - |
      LET combined_search = SELECT * FROM if(
        condition=(Use_Raw_NTFS = "Y"),
        then=ntfs_search,
        else=file_search)

    - |
      LET more_recent = SELECT * FROM if(
        condition=MoreRecentThan,
        then={
          SELECT * FROM combined_search
          WHERE Modified > atoi(string=MoreRecentThan)
        }, else=combined_search)

    - |
      SELECT * FROM if(
        condition=ModifiedBefore,
        then={
          SELECT * FROM more_recent
          WHERE Modified < atoi(string=ModifiedBefore)
        }, else=more_recent)
```
   {{% /expand %}}
