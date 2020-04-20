# Zero Knowledge Plugin

## Zero What ?
Bareos can encrypt your data before sending it to the Storage Daemon but the filenames still
remain as cleartext in the catalog (the sql database maintained by the Bareos director).

If your filenames contain sensible data (e.g. patients' names) you may hesitate to backup your data,
because sensible data goes into the database of the SQL server. If this is maintained by some
external providers, some users may require encryption here for compliance reasons.

This plugin encrypts filenames, directory names and even link sources and destination.

**NOTE**: you have to enable data-encryption itself to have the file contents encrypted. See https://docs.bareos.org/TasksAndConcepts/DataEncryption.html for more details

## Requirements
Needs the Python module *cryptography* installed in your Python2 environment.
It will also need the latest *BareosFdPluginBaseclass.py* and *BareosFdPluginLocalFileset.py*, currently in https://github.com/bareos/bareos/tree/dev/maik/master/pluginBaseClassLocalFileset


## Configuration
You need 2 parameters:
*filename* This is a file on the client, containing a list of file or directories to be backed up.
*keyfile* This is a file on the client, containing the encryption key in BASE64 format

Snippet for a fileset definition:

```
FileSet {
  Name = "zntest"
  Description = "fileset just to backup some files for selftest"
  Include {
    Options {
      Signature = MD5 # calculate md5 checksum per file
    }
   Plugin = "python:module_path=/usr/lib64/bareos/plugins:module_name=bareos-fd-zero-knowledge:filename=/etc/bareos/extra-files:keyfile=/etc/bareos/keyfile"
  }
}
```

You can create a random key like this:
```
echo -e "from cryptography.fernet import Fernet\nf=Fernet.generate_key()\nprint f" | python2.7
```

## Example
We just backup the /etc/vimrc, which we define in the one and only line in /etc/bareos/extra-files
We create a random key like above and redirect the output into our keyfile:
```
echo -e "from cryptography.fernet import Fernet\nf=Fernet.generate_key()\nprint f" | python2.7 > /etc/bareos/keyfile
```

We put the above defined fileset in a file in */etc/bareos/bareos-dir.d/fileset/zntest.conf* and define a job zntest in */etc/bareos/bareos-dir.d/job/zntest.conf*
```
Job {
  Name = "zntest"
  JobDefs = "DefaultJob"
  Client = "bareos-fd"
  Schedule = "Never"
  FileSet = "zntest"
}
```

Now we run a job:
```
echo "run job=zntest Level=Full yes" | bconsole
...
Job queued. JobId=636
```

The last line is our Jobid, in this case 636. Now let's see what files Bareos has in it's catalog:
```
[root@centacht bareos]# echo "list files jobid=636" | bconsole 
Connecting to Director localhost:9101
 Encryption: TLS_CHACHA20_POLY1305_SHA256
1000 OK: bareos-dir Version: 19.2.6 (11 February 2020)
bareos.org build binary
bareos.org binaries are UNSUPPORTED by bareos.com.
Get official binaries and vendor support on https://www.bareos.com
You are connected using the default console

Enter a period (.) to cancel a command.
list files jobid=636
Automatically selected Catalog: MyCatalog
Using Catalog "MyCatalog"
 gAAAAABena_0q0nr5izUt5u9VUBXPJ2kmmnHCkCFeePGDABX5Nppn_Xl3qIGwNvgJeiliTcMsEA-gWlSfS-mK0ZIfrsxj6GoSw==
You have messages.
```


