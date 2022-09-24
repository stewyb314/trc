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
the receivesd commands and stores the results.
### trc-client usage
**Design Consideration:** When possible I like outputting commands in JSON. For small blocks of data it is a good compromise between being
human-readable and machine parseable.

`trc-client` consists of the sub commands `start`, `stop` `status` and `output`. All the commands except `output` return
JSON. `output` streams the output of a command (see [output](#output) section below)The following options are common to
all subcommands:
```shell
  Options:

  -help
    	print help
  -host string
    	remote host to connect to (default "127.0.0.1")
  -ident string
    	config file with paths to ssl certs and keys (required)
  -port int
    	remote port to connect to (default 50051)

Where -ident is the path to a JSON file with information about SSL certs and keys:
{
	"ca-cert": "<path to self-signed CA certificate>",
	"public-key": "<path to host's public key>",
	"host-cert": "<path to host cert signed by 'ca-cert'">
}

```

More on the identy file in [Authentication](#authentication).

### subcommand error output
On error, all sub commands output:
```json
{
  "error": "error message",
  "code": "numerical error code"
}
```
### start subcommand
`start` starts a new shell command on the remote server.  

Usage: 
`trc-client start [-host|-port|-cert ] <command> [command arguments]`.

Output:
```json
{
  "id": "<unique UUID>",
  "error": "<null|error string>"
}
```

`id`: a unique identifier that can be used to stop or status commands
`error`: if non-null, the command could not be executed

### status subcommand
The `status` command retrieves information about a previously started command:

```shell
Usage: trc-client [options] status <command id>

Get status of a previously started command.
```

Output:
```json
{
  "id": "<command UUID>",
  "command": "<command run>",
  "args": [ 
            "command arg1",
            "command arg2"
  ],
  "state": "[running,completed,canceled]",
  "status": "<exit status of command>"
}
```

### stop subcommand

The `stop` commmand stops a running command.  If the command is not running, an error is returned.

```shell
Usage: trc-client [options] stop <command id>
```

Output:
```json
{
  "success": "true|false"
}
```


