---
title: Linux.Sys.LastUserLogin
hidden: true
tags: [Client Artifact]
---

Find and parse system wtmp files. This indicate when the user last logged in.

```yaml
name: Linux.Sys.LastUserLogin
description: Find and parse system wtmp files. This indicate when the
             user last logged in.
parameters:
  - name: wtmpGlobs
    default: /var/log/wtmp*

  - name: MaxCount
    default: 10000
    type: int64

export: |
  LET wtmpProfile <= '''
  [
    ["Header", 0, [
    
    ["records", 0, "Array", {
        "type": "utmp",
        "count": "x=>MaxCount",
        "max_count": "x=>MaxCount",
    }],
    ]],
    ["utmp", 384, [
        ["ut_type", 0, "Enumeration", {
            "type": "short int",
            "choices": {
               "0": "EMPTY",
               "1": "RUN_LVL",
               "2": "BOOT_TIME",
               "5": "INIT_PROCESS",
               "6": "LOGIN_PROCESS",
               "7": "USER_PROCESS",
               "8": "DEAD_PROCESS"
             }
          }],
        ["ut_pid", 4, "int"],
        ["ut_terminal", 8, "String", {"length": 32}],
        ["ut_terminal_identifier", 40, "String", {"length": 4}],
        ["ut_user", 44, "String", {"length": 32}],
        ["ut_hostname", 76, "String", {"length": 256}],
        ["ut_termination_status", 332, "int"],
        ["ut_exit_status", 334, "int"],
        ["ut_session", 336, "int"],
        ["ut_timestamp", 340, "int32"],
        ["ut_ip_address", 348, "int64"],
    ]
    ]
    ]]
    ]'''
    
sources:
  - precondition: |
      SELECT OS From info() where OS = 'linux'
    query: |
      LET parsed = SELECT FullPath, parse_binary(
                   filename=FullPath,
                   profile=wtmpProfile,
                   struct="Header"
                 ) AS Parsed
      FROM glob(globs=split(string=wtmpGlobs, sep=","))
      SELECT * FROM foreach(row=parsed,
      query={
         SELECT * FROM foreach(row=Parsed.records,
         query={
           SELECT FullPath, ut_type AS Type,
              ut_id AS ID,
              ut_pid as PID,
              ut_hostname as Host,
              ut_user as User,
              ip(netaddr4_le=ut_ip_address) AS IpAddr,
              ut_terminal as Terminal,
              timestamp(epoch=ut_timestamp) as login_time
          FROM scope()
        })
      }) WHERE Type != "EMPTY" AND PID != 0

```
