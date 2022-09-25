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

The client and server communicate over gRPC.

## trc-client usage
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

Each user has a unique identity config file.  Besides authentication, the server uses this information to distinguish between users.
More on the identy file in [Authentication](#Authentication-and-Authorization).

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

```
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

### output subcommand
The `output` subcommand streams results from a command.  If the command is still running on the remote machine, the output will be live streamed until
 the command finishes or is stopped by another trc command. If the command is not running, it will exit after all lines have been dispalyed
 
 ```
 Usage: trc-client [options] status <command id>
 Stream the output of a command
```

Output:
```
command output line1
command output line2
command output line3
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

## trc-server usage
The server only takes a single option:
```
Options:
    - auth-config
        configuration file of allowed host certificates

File format:

{
     "ca-cert": "<path to CA cert>",
     "server-key": "<path to server public key>",
     "server-cert": "<path to server's cert signed by CA cert>",
     "client-certs": [
     	"path-to-cert1",
	"path-to-cert2",
	...
     ]
}
```
Where client-certs is a list of client certificates allowed to connect to the server. 

## Authentication and Authorization

Authentication and Authorization takes place in three steps:

1. Verify the client certificate is signed by the trusted CA.  This step is handled by gRPC library
2. Verify the client certificate is known to the server.  This is determined by checking the "client-certs" entry in the auth-config file passed into the server at startup.  Each cert is loaded into memory and the public key from the request is comapred to the public key of the certificates.  If a match is found, this step passes.
3. For `status`, `stop` and `output` sub-commands, verify the client which started the command is same client attempting to access the command (ie user 1 starts the command, user 2 doesn't have permissions to status the command)

More details in [Authentication Service](#Authentication Service)

## Server Design
```
                        +-----------------------------------------------------------------+
                        |           ┌────────┐            ┌────────────┐                  |
                        |           │Auth.   |            |   Job      |                  |
                        |           │Service |            |  Repository|                  |
                        |           └──-─┬───┘            └────────────┘                  |
                        |                |                       |                        |
                        |                |                       |              ┌───────┐ |
                        |                |                       |        +-----| Job 1 | |
                        |                |                       |        |     └───────┘ |
     ┌──────┐           |            ┌───┴──┐            ┌───────┴───┐    |               |
     │Client│-----------+------------│ API  │------------│Job Manager│----+----┌───────┐  |
     └──-───┘           |            └──────┘            └───────────┘    |    | Job 2 |  |
                        |                                                 |    └───────┘  |
                        |                                                 |               |
                        |                                                 |               |
                        |                                                 |    ┌────────┐ |
                        |                                                 +----| Job N  | |
                        |                                                      └────────┘ |
                        |                                                                 |
                        +------------------------------------------------------------------+
```

**Terminology** A `job` in this context consists of a single command and associated metadata.  

**Design Considerations** For the sake of simplicity, data is NOT persisted between server restarts.  Job Repository interface  will provide 
CRUD operations so changing the implementation be backed by a database won't require changes to the interface.

## API
The API receives the incoming request from a client.  It authenticates the request and calls the appropriate method in the Job Manager.

## Auth Service
The auth service is responsible for verifying the connected client has permissions to run commands on the server.  On startup, the auth service reads in 
a list of authorized client certificates.  When a request comes in, the public key is obtained from the request and passed to Auth Service.  If the key
matches one of the known certificates, a unique user ID is returned.  The ID is then attached to each job as they are started.  The user ID then used to
prevent a user from getting a different user's command info.

Because data is not presisted across server restarts, user IDs aren't guaranteed to be consistent across restarts.  They will, however, be consistent on 
individual runs of the server.  In a production system, the Auth Service would be backed by a database and consistency maintained.

## Job Repository
The job repository preserves the state of each job.  For this implementation it is not backed by a database, but the schema is:

```
     command_id        |     user_id        |    state     |   command     |    args       |    output_file
   --------------------+--------------------+--------------+---------------+---------------+-------------------------
   UUID of the command | ID of user who     | state of the | command which | list of args  | file with command output
   which was run.      | started command.   | command.     | was run.      | to the command|
```
## Job Manager
The job manager is responsible for managing jobs.  The job manager provdes the interface to start, stop, status and get output of jobs.  
As each job is started, it sets up the cgroup by creating the directoy and setting up the appropriate files.  It then manages the lifecycle of the job,
both notifying the job if the client stops the job and receiving notification as jobs complete.  As the job changes state, the job manager updates the job
repository.

As jobs complete, they are removed from the job manager, with the results stored in the job repository.

## Jobs
Jobs represent a single command launched by the client.  Besides monitoring the job for completion, the job is also responsible for providing a streaming
service to clients.   The job accomplishes this by writing the output of each command to a file.  As clients execute the `output` subcommand, a watcher is
created.  Each watcher essentially tails the output file and writes the result to a channel.  When the command completes (either by finishing execution or
by client stopping command) the watcher is notified via a channel.

In psudo code the watcher tailing an output file:
```
    while true {
    	line = readLineFromOutputFile()
	if nothing to read {
	    done = readFromDoneChannel()
	    if done {
	        return
	    }
	    sleep(1)
	}
	else {
	    writeOutputChannel(line)
	}
    }
```

By implementing the streaming using files, the file offsets are managed by the OS, rather than the job.  This significantly simplifies the design.

## cgroups

All jobs will run in a unique cgroup.  This will be setup by the job manager when the job is started.  When the job exists, the job manager will clean up 
up the cgroup directories.











