---
title: Windows.Detection.EnvironmentVariables
hidden: true
tags: [Client Artifact]
---

Find processes with the specified environment variables.


```yaml
name: Windows.Detection.EnvironmentVariables
description: |
   Find processes with the specified environment variables.

parameters:
   - name: ProcessNameRegex
     default: .
     type: regex
   - name: PidRegex
     default: .
     type: regex
   - name: EnvironmentVariableRegex
     default: COMSPEC|COR_PROFILER
     type: regex
   - name: FilterValueRegex
     default: .
     type: regex
   - name: WhitelistValueRegex
     description: Ignore these values
     default: ^C:\\Windows\\.+cmd.exe$
     type: regex

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'

    query: |
      SELECT * FROM foreach(
      row={
          SELECT * FROM Artifact.Windows.Memory.ProcessInfo(
             ProcessNameRegex=ProcessNameRegex, PidRegex=PidRegex)
      },
      query={
          SELECT Pid, Name, ImagePathName, CommandLine,
             _key AS Var, _value AS Value
          FROM items(item=Env)
      })
      WHERE Var =~ EnvironmentVariableRegex
        AND Value =~ FilterValueRegex
        AND NOT Value =~ WhitelistValueRegex

    notebook:
      - type: Markdown
        template: |-
          # Process Environment Variables

          Environment variables control the way subprocesses work. In
          this artifact we look for processes with unusual sets of
          environment variables.

          {{ $unusual := Query "SELECT * FROM source() WHERE \
              Var =~ 'COR_PROFILER|COMPlus_ETWEnabled'" | Expand }}

          {{ if $unusual }}
          ## Some unusual environment variables.

          There have been some unusual environment variables
          detected. These normally indicate malicious activity.

          {{ Table $unusual }}

          {{ end }}

          {{ $unusual = Query "SELECT * FROM source() WHERE \
              Var =~ 'COMSPEC' AND NOT Value =~ 'cmd.exe$'" | Expand }}
          {{ if $unusual }}

          ## Unusual COMSPEC setting.

          The `COMSPEC` environment variable is usually used to launch
          the command prompt (cmd.exe) but Velociraptor found some
          hits where this is not the case. It could indicate malicious
          activity.

          {{ Table $unusual }}

          {{ end }}

      - type: VQL
        template: |

          /* Markdown
          ## All collected results.

          */

          SELECT * FROM source()
          LIMIT 50

```
