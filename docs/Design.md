# TRC Design
Teleport Remote Command

## What
The teleport remote command (trc) provides a means to execute remote commands on a Linux server.

## Why
trc will allow Teleport support techs to log into to customer's Linux servers and execute shell commands to troubleshoot 
customer issues.

## Details
trc is split into two executables: `trc-server` and `trc-client`.  `trc-client` runs in the Teleport environment and 
sends arbitrary shell commands to `trc-server`, which is installed on the customers' Linux box. `trc-server` is responsible for executing
the receives the command, run it and stores the results.
### trc-client usage
**Design Consideration:** When possible I like outputting commands in JSON.  It is a good compromise between being
human-readable and machine parseable.

`trc-client` consists of the sub commands `start`, `stop` `status` and `output`. All the commands except `output` return
JSON. `output` streams the output of a command (see [output](#output) section below)The following options are common to
all subcommands:
```shell
  -host   remote hostname or IP address to connect to.  Default is localhost
  -port   remote port to connect to.  Default is XXXX
  -cert   path to ssl certificate to authenticate with Default ./cert/trc-client.crt
```
### start subcommand
`start` starts a new shell command on the remote server.  

Usage: 
`trc-client start [-host|-port|-cert ] <command> [command arguments]`.

Where command is the command to be run, followed by the command arguments

Example:
`trc-client start -host my-host ls -l /tmp`

Will get a long listing of the /tmp directory

Everything after the connection options is assumed to be a part of the command.

The output of this command is:
```json
{
  "id": "<unique UUID>",
  "error": "<null|error string>"
}
```

`id`: a unique identifier that can be used to stop or status commands
`error`: if non-null, the command could not be executed

### status subcommand
The `status` sub-command gets the status of a command. The



