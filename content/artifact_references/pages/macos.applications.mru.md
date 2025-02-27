---
title: MacOS.Applications.MRU
hidden: true
tags: [Client Artifact]
---

Parse the MRU from MacOS users


```yaml
name: MacOS.Applications.MRU
description: |
   Parse the MRU from MacOS users

reference:
  - https://mac-alias.readthedocs.io/en/latest/bookmark_fmt.html
  - https://github.com/al45tair/mac_alias
  - https://www.mac4n6.com/blog/2016/7/10/new-script-macmru-most-recently-used-plist-parser

type: CLIENT

parameters:
   - name: FinderPlistPath
     default: /Users/*/Library/Preferences/com.apple.finder.plist

export: |
        -- Parser for MAC Bookmark format
        LET type_lookup <= dict(
           `0x100`="__DataString",
           `0x200`="__DataData",
           `0x300`="__DataUint32",
           `0x400`="__DataDate",
           `0x500`="__DataBool",
           `0x600`="__DataArray",
           `0x700`="__DataDict",
           `0x800`="__DataUUID",
           `0x900`="__DataURL"
           )

        LET MRULookup <= dict(
           `0x2040`="Volume Bookmark",
           `0x2002`="Volume Path",
           `0x2020`="Volume Flags",
           `0x2030`="Volume is Root FS",
           `0x2011`="Volume UUID",
           `0x2012`="Volume Size",
           `0x2013`="Volume Creation Date",
           `0x2005`="Volume URL",
           `0x2040`="Volume Bookmark",
           `0x2050`="Volume Mount Point",
           `0xf080`="Security Extension",
           `0xf081`="Security Extension",
           `0x1004`="Target Path",
           `0x1005`="Target CNID Path",
           `0xc001`="Containing Folder Index",
           `0x1040`="Target Creation Date",
           `0x1010`="Target Flags",
           `0x1020`="Target Filename",
           `0xc011`="Creator Username",
           `0xc012`="Creator UID"
        )

        LET BookmarkProfile = '''[
         ["Header", 0, [
          ["Magic", 0, "String", {
              length: 4,
          }],
          ["Size", 4, "uint32"],
          ["HeaderSize", 12, "uint32"],
          ["TOCOffset", "x=>x.HeaderSize", "uint32"],
          ["TOC", "x=>x.TOCOffset + x.HeaderSize", "TOC"]
         ]],
         ["TOC", 0, [
          ["SizeOfTOC", 0, "uint32"],
          ["Magic", 4, "uint32"],
          ["TOCId", 8, "uint32"],
          ["NextTOC", 12, "uint32"],
          ["TOCCount", 16, "uint32"],
          ["Items", 20, "Array", {
              type: "TOCItem",
              count: "x=>x.TOCCount",
          }]
         ]],
         ["__TOCArrayPtr", 4, [
          ["Offset", 0, "uint32"],
          ["Item", 0, "Profile", {
            type: "TOCValue",
            offset: "x=>x.Offset + 48"
           }]
         ]],
         ["TOCValue", 0, [
           ["MyOffset", 0, "Value", {
               value: "x=>x.StartOf",
           }],
           ["length", 0, "uint32"],
           ["subtype", 4, "BitField", {
               type: "uint32",
               start_bit: 0,
               end_bit: 8,
            }],
            ["data_type", 4, "BitField", {
               type: "uint32",
               start_bit: 8,
               end_bit: 32,
            }],
            ["data", 0, "Value", {
               value: "x=>get(item=x, field=get(item=type_lookup, field=format(format='%#x', args=x.data_type)))",
            }],
            ["__DataString", 8, "String", {
               length: "x=>x.length",
               term: "",
            }],
            ["__DataData", 0, "Value", {
               value: "x=>format(format='%x', args=x.__DataStr)",
            }],
            ["__DataDateFloat", 8, "float64be"],
            ["__DataDate", 0, "Value", {
               value: "x=>timestamp(cocoatime=x.__DataDateFloat)",
            }],
            ["__DataUint32", 8, "uint32"],
            ["__DataBool", 0, "Value", {
                value: "x=>if(condition=x.subtype, then=TRUE, else=FALSE)",
            }],
            ["__DataURL", 0, "Value", {
               value: "x=>x.__DataString",
            }],
            ["__DataArrayOffsets", 8, "Array", {
               count: "x=>x.length / 4",
               type: "__TOCArrayPtr"
            }],
            ["__DataArray", 0, "Value", {
               value: "x=>x.__DataArrayOffsets.Item.data",
            }],
         ]],
         ["TOCItem", 12, [
           ["ID", 0, "uint32"],
           ["Offset", 4, "uint32"],
           ["TOCValue", "x=>x.Offset + 48 - x.StartOf", "TOCValue"],
         ]]
        ]
        '''

        LET ParseBookmark(Bookmark) =
           SELECT _value.name AS Name,
                  get(item=MRULookup, field=format(format="%#x", args=ID)) AS Field,
                  format(format="%#x", args=ID) AS FieldID,
                  format(format="%#x", args=TOCValue.data_type) AS data_type,
                  regex_replace(re="__Data", replace="",
                        source=get(item=type_lookup,
                        field=format(format="%#x",
                              args=TOCValue.data_type))) AS type,
                  TOCValue.data AS data

           FROM foreach(row=parse_binary(
                        accessor="data", filename=Bookmark,
                        profile=BookmarkProfile, struct="Header").TOC.Items)

sources:
  - query: |
        -- Parse the Plist file
        SELECT * FROM foreach(row={
          SELECT FullPath FROM glob(globs=FinderPlistPath)
        }, query={
          SELECT * FROM foreach(row={
            SELECT FXRecentFolders FROM plist(file=FullPath)
          }, query={
            SELECT *
            FROM foreach(row=FXRecentFolders, query={
               SELECT *, FullPath
               FROM ParseBookmark(Bookmark=_value.`file-bookmark`)
            })
          })
        })

```
