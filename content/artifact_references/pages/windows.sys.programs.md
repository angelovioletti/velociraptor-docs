---
title: Windows.Sys.Programs
hidden: true
tags: [Client Artifact]
---

Represents products as they are installed by Windows Installer. A product generally
correlates to one installation package on Windows. Some fields may be blank as Windows
installation details are left to the discretion of the product author.

Limitations: This key parses the live registry hives - if a user is not logged in then their data will not be resident in HKU and therefore you should parse the hives on disk (including within VSS/Regback).


```yaml
name: Windows.Sys.Programs
description: |
  Represents products as they are installed by Windows Installer. A product generally
  correlates to one installation package on Windows. Some fields may be blank as Windows
  installation details are left to the discretion of the product author.

  Limitations: This key parses the live registry hives - if a user is not logged in then their data will not be resident in HKU and therefore you should parse the hives on disk (including within VSS/Regback).

reference:
  - https://github.com/facebook/osquery/blob/master/specs/windows/programs.table

parameters:
  - name: programKeys
    default: >-
      HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*,
      HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*,
      HKEY_USERS\*\Software\Microsoft\Windows\CurrentVersion\Uninstall\*

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'
    queries:
      - |
        SELECT Key.Name as KeyName,
               Key.Mtime AS KeyLastWriteTimestamp,
               DisplayName,
               DisplayVersion,
               InstallLocation,
               InstallSource,
               Language,
               Publisher,
               UninstallString,
               InstallDate,
               Key.FullPath as KeyPath
        FROM read_reg_key(globs=split(string=programKeys, sep=',[\\s]*'),
                          accessor="registry")

```
