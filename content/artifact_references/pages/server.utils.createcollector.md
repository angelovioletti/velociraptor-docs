---
title: Server.Utils.CreateCollector
hidden: true
tags: [Server Artifact]
---

A utility artifact to create a stand alone collector.

This artifact is actually invoked by the Offline collector GUI and
that is the recommended way to launch it. You can find the Offline
collector builder in the `Server Artifacts` section of the GUI.


```yaml
name: Server.Utils.CreateCollector
description: |
  A utility artifact to create a stand alone collector.

  This artifact is actually invoked by the Offline collector GUI and
  that is the recommended way to launch it. You can find the Offline
  collector builder in the `Server Artifacts` section of the GUI.

type: SERVER

parameters:
  - name: OS
    default: Windows
    type: choices
    choices:
      - Windows
      - Windows_x86
      - Linux
      - MacOS

  - name: artifacts
    description: A list of artifacts to collect
    type: json_array
    default: |
      ["Generic.Client.Info"]

  - name: encryption_scheme
    description: |
      Encryption scheme to use. Currently supported are Password, X509 or PGP

  - name: encryption_args
    description: |
      Encryption arguments
    type: json
    default: |
      {}

  - name: parameters
    description: A dict containing the parameters to set.
    type: json
    default: |
      {}

  - name: target
    description: Output type
    type: choices
    default: ZIP
    choices:
      - ZIP
      - GCS
      - S3
      - SFTP
      - Azure
      - SMBShare

  - name: target_args
    description: Type Dependent args
    type: json
    default: "{}"

  - name: opt_verbose
    default: Y
    type: bool
    description: Show verbose progress.

  - name: opt_banner
    default: Y
    type: bool
    description: Show Velociraptor banner.

  - name: opt_prompt
    default: N
    type: bool
    description: Wait for a prompt before closing.

  - name: opt_admin
    default: Y
    type: bool
    description: Require administrator privilege when running.

  - name: opt_tempdir
    default:
    description: A directory to write tempfiles in

  - name: opt_level
    default: "4"
    type: int
    description: Compression level (0=no compression).

  - name: opt_format
    default: "jsonl"
    description: Output format (jsonl or csv)

  - name: opt_output_directory
    default: ""
    description: An optional output directory prefix

  - name: opt_cpu_limit
    default: "0"
    type: int
    description: |
      A number between 0 to 100 representing the target maximum CPU
      utilization during running of this artifact.

  - name: opt_progress_timeout
    default: "1800"
    type: int
    description: |
      If specified the collector is terminated if it made no progress
      in this long. Note: Execution time may be a lot longer since
      each time any result is produced this counter is reset.

  - name: opt_timeout
    default: "0"
    type: int
    description: |
      If specified the collection must complete in the given time. It
      will be cancelled if the collection exceeds this time.

  - name: StandardCollection
    type: hidden
    default: |
      LET _ <= log(message="Will collect package " + filename)

      SELECT * FROM collect(artifacts=Artifacts,
            args=Parameters, output=filename + ".zip",
            cpu_limit=CpuLimit,
            progress_timeout=ProgressTimeout,
            timeout=Timeout,
            password=pass[0].Pass,
            level=Level,
            format=Format,
            metadata=ContainerMetadata)

  - name: S3Collection
    type: hidden
    default: |
      // A utility function to upload the file.
      LET upload_file(filename, name, accessor) = upload_s3(
          file=filename,
          accessor=accessor,
          bucket=TargetArgs.bucket,
          name=name,
          credentialskey=TargetArgs.credentialsKey,
          credentialssecret=TargetArgs.credentialsSecret,
          region=TargetArgs.region,
          endpoint=TargetArgs.endpoint,
          serversideencryption=TargetArgs.serverSideEncryption,
          noverifycert=TargetArgs.noverifycert)

  - name: GCSCollection
    type: hidden
    default: |
      // A utility function to upload the file.
      LET upload_file(filename, name, accessor) = upload_gcs(
          file=filename,
          accessor=accessor,
          bucket=TargetArgs.bucket,
          project=GCSBlob.project_id,
          name=name,
          credentials=TargetArgs.GCSKey)

  - name: AzureSASURL
    type: hidden
    default: |
      // A utility function to upload the file.
      LET upload_file(filename, name, accessor) = upload_azure(
          file=filename,
          accessor=accessor,
          sas_url=TargetArgs.sas_url,
          name=name)

  - name: SMBCollection
    type: hidden
    default: |
      // A utility function to upload the file.
      LET upload_file(filename, name, accessor) = upload_smb(
          file=filename,
          accessor=accessor,
          username=TargetArgs.username,
          password=TargetArgs.password,
          server_address=TargetArgs.server_address,
          name=name)

  - name: SFTPCollection
    type: hidden
    default : |
      LET upload_file(filename, name, accessor) = upload_sftp(
        file=filename,
        accessor=accessor,
        name=name,
        user=TargetArgs.user,
        path=TargetArgs.path,
        privatekey=TargetArgs.privatekey,
        endpoint=TargetArgs.endpoint,
        hostkey = TargetArgs.hostkey)

  - name: CommonCollections
    type: hidden
    default: |
      // Add all the tools we are going to use to the inventory.
      LET _ <= SELECT inventory_add(tool=ToolName, hash=ExpectedHash)
       FROM parse_csv(filename="/uploads/inventory.csv", accessor="me")
       WHERE log(message="Adding tool " + ToolName)

      LET baseline <= SELECT Fqdn, basename(path=Exe) AS Exe FROM info()

      // Make the filename safe on windows but we trust the OutputPrefix.
      LET filename <= OutputPrefix + regex_replace(
          source=format(format="Collection-%s-%s",
                        args=[baseline[0].Fqdn,
                              timestamp(epoch=now()).MarshalText]),
          re="[^0-9A-Za-z\\-]", replace="_")

      -- Make a random hex string as a random password
      LET RandomPassword <= SELECT format(format="%02x",
            args=rand(range=255)) AS A
      FROM range(end=25)

      LET pass = SELECT * FROM switch(a={

         -- For X509 encryption we use a random session password.
         SELECT join(array=RandomPassword.A) as Pass From scope()
         WHERE encryption_scheme =~ "pgp|x509"
          AND log(message="I will generate a container password using the %v scheme",
                  args=encryption_scheme)

      }, b={

         -- Otherwise the user specified the password.
         SELECT encryption_args.password as Pass FROM scope()
         WHERE encryption_scheme =~ "password"

      }, c={

         -- No password specified.
         SELECT Null as Pass FROM scope()
      })

      -- For X509 encryption_scheme, store the encrypted
      -- password in the metadata file for later retrieval.
      LET ContainerMetadata = if(
          condition=encryption_args.public_key,
          then=dict(
             EncryptedPass=pk_encrypt(data=pass[0].Pass,
                public_key=encryption_args.public_key,
             scheme=encryption_scheme),
          Scheme=encryption_scheme,
          PublicKey=encryption_args.public_key))

  - name: CloudCollection
    type: hidden
    default: |
      LET TargetArgs <= target_args

      // Try to upload the log file now to see if we are even able to
      // upload at all - we do this to avoid having to collect all the
      // data and then failing the upload step.
      LET _ <= log(message="Uploading to " + filename + ".log")
      LET upload_test <= upload_file(
          filename="Test upload from " + baseline[0].Exe,
          accessor="data",
          name=filename + ".log")

      LET _ <= log(message="Will collect package " + filename +
         " and upload to cloud bucket " + TargetArgs.bucket)

      LET collect_and_upload = SELECT
          upload_file(filename=Container,
                      name=filename+".zip",
                      accessor="file") AS Upload,
          upload_file(filename=baseline[0].Exe + ".log",
                      name=filename+".log",
                      accessor="file") AS LogUpload

      FROM collect(artifacts=Artifacts,
          args=Parameters,
          format=Format,
          output=tempfile(extension=".zip"),
          cpu_limit=CpuLimit,
          progress_timeout=ProgressTimeout,
          timeout=Timeout,
          password=pass[0].Pass,
          level=Level,
          metadata=ContainerMetadata)

      SELECT * FROM if(condition=upload_test.Path,
          then=collect_and_upload,
          else={SELECT log(
             message="Aborting collection: Failed to upload to cloud bucket!")
          FROM scope()})

  - name: FetchBinaryOverride
    type: hidden
    description: |
       A replacement for Generic.Utils.FetchBinary which
       grabs files from the local archive.

    default: |
       LET RequiredTool <= ToolName

       LET matching_tools <= SELECT ToolName, Filename
       FROM parse_csv(filename="/uploads/inventory.csv", accessor="me")
       WHERE RequiredTool = ToolName

       LET get_ext(filename) = parse_string_with_regex(
             regex="(\\.[a-z0-9]+)$", string=filename).g1

       LET temp_binary <= if(condition=matching_tools,
       then=tempfile(
                extension=get_ext(filename=matching_tools[0].Filename),
                remove_last=TRUE,
                permissions=if(condition=IsExecutable, then="x")))

       SELECT copy(filename=Filename, accessor="me", dest=temp_binary) AS FullPath,
              Filename AS Name
       FROM matching_tools

sources:
  - query: |
      LET Binaries <= SELECT * FROM foreach(
          row={
             SELECT tools FROM artifact_definitions(deps=TRUE, names=artifacts)
          }, query={
             SELECT * FROM foreach(row=tools,
             query={
                SELECT name AS Binary FROM scope()
             })
          }) GROUP BY Binary

      // Choose the right target binary depending on the target OS
      LET tool_name = SELECT * FROM switch(
       a={ SELECT "VelociraptorWindows" AS Type FROM scope() WHERE OS = "Windows"},
       b={ SELECT "VelociraptorWindows_x86" AS Type FROM scope() WHERE OS = "Windows_x86"},
       c={ SELECT "VelociraptorLinux" AS Type FROM scope() WHERE OS = "Linux"},
       d={ SELECT "VelociraptorDarwin" AS Type FROM scope() WHERE OS = "MacOS"},
       e={ SELECT "" AS Type FROM scope()
           WHERE NOT log(message="Unknown target type " + OS) }
      )

      LET Target <= tool_name[0].Type

      // This is what we will call it.
      LET CollectorName <= format(
          format='Collector_%v',
          args=inventory_get(tool=Target).Definition.filename)

      LET CollectionArtifact <= SELECT Value FROM switch(
        a = { SELECT CommonCollections + StandardCollection AS Value
              FROM scope()
              WHERE target = "ZIP" },
        b = { SELECT S3Collection + CommonCollections + CloudCollection AS Value
              FROM scope()
              WHERE target = "S3" },
        c = { SELECT GCSCollection + CommonCollections + CloudCollection AS Value
              FROM scope()
              WHERE target = "GCS" },
        d = { SELECT SFTPCollection + CommonCollections + CloudCollection AS Value
              FROM scope()
              WHERE target = "SFTP" },
        e = { SELECT AzureSASURL + CommonCollections + CloudCollection AS Value
              FROM scope()
              WHERE target = "Azure" },
        f = { SELECT SMBCollection + CommonCollections + CloudCollection AS Value
              FROM scope()
              WHERE target = "SMBShare" },
        z = { SELECT "" AS Value  FROM scope()
              WHERE log(message="Unknown collection type " + target) }
      )

      LET use_server_cert = encryption_scheme =~ "x509"
         AND NOT encryption_args.public_key =~ "----BEGIN CERTIFICATE-----"
         AND log(message="Pubkey encryption specified, but no cert/key provided. Defaulting to server frontend cert")

      -- For x509, if no public key cert is specified, we use the
      -- server's own key. This makes it easy for the server to import
      -- the file again.
      LET updated_encryption_args <= if(
         condition=use_server_cert,
         then=dict(public_key=server_frontend_cert(),
                   scheme="x509"),
         else=encryption_args
      )

      -- Add custom definition if needed. Built in definitions are not added
      LET definitions <= SELECT * FROM chain(
      a = { SELECT name, description, tools, parameters, sources
            FROM artifact_definitions(deps=TRUE, names=artifacts)
            WHERE NOT built_in AND
              log(message="Adding artifact_definition for " + name) },

      // Create the definition of the Collector artifact.
      b = { SELECT "Collector" AS name, (
                    dict(name="Artifacts",
                         default=serialize(format='json', item=artifacts),
                         type="json_array"),
                    dict(name="Parameters",
                         default=serialize(format='json', item=parameters),
                         type="json"),
                    dict(name="encryption_scheme", default=encryption_scheme),
                    dict(name="encryption_args",
                         default=serialize(format='json', item=updated_encryption_args),
                         type="json"
                         ),
                    dict(name="Level", default=opt_level, type="int"),
                    dict(name="Format", default=opt_format),
                    dict(name="OutputPrefix", default=opt_output_directory),
                    dict(name="CpuLimit", type="int",
                         default=opt_cpu_limit),
                    dict(name="ProgressTimeout", type="int",
                         default=opt_progress_timeout),
                    dict(name="Timeout", default=opt_timeout, type="int"),
                    dict(name="target_args",
                         default=serialize(format='json', item=target_args),
                         type="json"),
                ) AS parameters,
                (
                  dict(query=CollectionArtifact[0].Value),
                ) AS sources
            FROM scope() },

      // Override FetchBinary to get files from the executable.
      c = { SELECT "Generic.Utils.FetchBinary" AS name,
            (
               dict(name="SleepDuration", type="int", default="0"),
               dict(name="ToolName"),
               dict(name="ToolInfo"),
               dict(name="IsExecutable", type="bool", default="Y"),
            ) AS parameters,
            (
               dict(query=FetchBinaryOverride),
            ) AS sources FROM scope()  }
      )

      LET optional_cmdline = SELECT * FROM chain(
        a={ SELECT "-v" AS Opt FROM scope() WHERE opt_verbose},
        b={ SELECT "--nobanner" AS Opt FROM scope() WHERE NOT opt_banner},
        c={ SELECT "--require_admin" AS Opt FROM scope() WHERE opt_admin},
        d={ SELECT "--prompt" AS Opt FROM scope() WHERE opt_prompt},
        e={ SELECT "--tempdir" AS Opt FROM scope() WHERE opt_tempdir},
        f={ SELECT opt_tempdir AS Opt FROM scope() WHERE opt_tempdir}
      )

      // Build the autoexec config file depending on the user's
      // collection type choices.
      LET autoexec <= dict(autoexec=dict(
          argv=("artifacts", "collect", "Collector",
                "--logfile", CollectorName + ".log") + optional_cmdline.Opt,
          artifact_definitions=definitions)
      )

      // Do the actual repacking.
      SELECT repack(
           upload_name=CollectorName,
           target=tool_name[0].Type,
           binaries=Binaries.Binary,
           config=serialize(format='json', item=autoexec))
      FROM scope()

```
