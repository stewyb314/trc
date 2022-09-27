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
the received commands and stores the results.

**Design Consideration** All commands are run as root and there are zero guard rails. If a user decides to get cute and run `rm -rf /`, then RIP their
machine. A production version of this would add protections like run as non-root or chroot'd to a specific directory.

The client and server communicate over gRPC.

## trc-client usage
**Design Consideration:** When possible I like outputting commands in JSON. For small blocks of data it is a good compromise between being
human-readable and machine parseable.

`trc-client` consists of the sub commands `start`, `stop` `status` and `output`. All the commands except `output` return
JSON. `output` streams the output of a command (see [output](#output-subcommand) section below)The following options are common to
all subcommands:
```
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
More on the identity file in [Authentication](#Authentication-and-Authorization).

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
`trc-client start [-host|-port|-ident] <command> [command arguments]`.

Output:
```json
{
  "id": "<unique UUID>"
}
```

`id`: a unique identifier that can be used to stop or status commands


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
}
```

### output subcommand
The `output` subcommand returns output of a command.  If the command is still running on the remote machine, the output will be live streamed until
 the command finishes or is stopped by another trc command. If the command is not running, it will exit after all lines have been displayed.
 
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

The `stop` command stops a running command.  If the command is not running, an error is returned.

```shell
Usage: trc-client [options] stop <command id>
```

Output:
```json
{
  "success": "true|false"
}
```


## gRPC protocol

The protobuf message definitions for sending and receiving commands between the client and server:
```
service TrcServer{
    rpc Start(StartRequest) returns (StartResponse);
    rpc Output(IdRequest) returns (stream OutputResponse);
    rpc Status(IdRequest) returns (StatusResponse);
    rpc Stop(IdRequest) returns (StopResponse);
}

message StartRequest {
    string command = 1;
}

message StartResponse {
    UUID id = 1;
    Error error = 2;
}

message IdRequest {
    UUID id = 1;
}

message OutputResponse {
    string output = 1;
    Error error = 2;
}


message StatusResponse {
    UUID id = 1;
    string cmd = 2;
    repeated string args = 3;
    string state = 4;
    Error error = 5;
}

message StopResponse {
    UUID id = 1;
    Error error = 2;
}

enum State {
    RUNNING = 0;
    COMPLETE = 1;
    STOPPED = 2;
    ERROR = 3;
}

message UUID {
    string value = 1;
}

message Error {
    int32 code = 1;
    string message = 2;
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
2. Verify the client certificate is known to the server.  This is determined by checking the "client-certs" entry in the auth-config file passed into the server at startup.  Each cert is loaded into memory and the public key from the request is compared to the public key of the certificates.  If a match is found, this step passes.
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
     │Client│-----------+------------│ API  │------------│Job Service│----+----┌───────┐  |
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
CRUD operations so changing the implementation to be backed by a database won't require changes to the interface.

## API
The API receives the incoming request from a client.  It authenticates the request and calls the appropriate method in the Job Service.

## Auth Service
The auth service is responsible for verifying the connected client has permissions to run commands on the server.  On startup, the auth service reads in 
a list of authorized client certificates.  When a request comes in, the public key is obtained from the request and passed to Auth Service.  If the key
matches one of the known certificates, a unique user ID is returned.  The ID is then attached to each job as they are started and used to authorize access
to a job (ie: if user 1 starts a job, only user 1 should have permission to status or stop the job).


## Job Repository
The job repository preserves the state of each job.  For this implementation it is not backed by a database, but the schema is:

```
     command_id        |     user_id        |    state     |   command     |    args       |    output_file
   --------------------+--------------------+--------------+---------------+---------------+-------------------------
   UUID of the command | ID of user who     | state of the | command which | list of args  | file with command output
   which was run.      | started command.   | command.     | was run.      | to the command|
```

The job repository provides CRUD access to the data (technially CRU operations, as delete is not a requirement of this project).


## Job Service
The job service is responsible for managing jobs.  The job service provides the interface to start, stop, status and get output of jobs.  
As each job is started, it sets up the cgroup by creating the directory and setting up the appropriate files.  It then manages the lifecycle of the job,
both notifying the job if the client stops the job and receiving notification as jobs complete.  As the job changes state, the job service updates the job
repository.

As jobs complete, they are removed from the job service, with the results stored in the job repository.

## Jobs
Jobs represent a single command launched by the client.  Besides monitoring the job for completion, the job is also responsible for providing a streaming
service to clients when displaying output of a running process.   The job accomplishes this by writing the output of each command to a file.  
As clients execute the `output` subcommand, a watcher is
created.  Each watcher essentially tails the output file and writes the result to a channel.  When the command completes (either by finishing execution or
by client stopping command) the watcher is notified via a channel.

In pseudo code the watcher tailing an output file:
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

All jobs will run in a unique cgroup.  This will be setup by the job service when the job is started.  When the job exists, the job service will clean up 
up the cgroup directories.

For this project, the cgroup values will be hard coded.  

**Design Consideration** There are two ways to add a process to a cgroup: either by starting the process using the `cexec` command or by placing the running processes' PID in the cgroup.procs file.  Using cexec command has the advatage of starting the process in the cgroup, however the command is not installed on some Linux distros by default.  Adding the pid has the advatange of not adding additional dependencies, but the disadvatage of starting outside of the cgroup constratings, albeit only briefly.  For this project, I opted to put the PID in the cgroup file.


## Workflow

All of the sub commands follow similar workflows, so as an example, this is the workflow for for `start`:

```
    ┌─────┐                 ┌─────┐                   ┌─────────┐                 ┌───────────┐
    | API |                 | Auth|                   | Job.    |                 |   Job     |
    └──┬──┘                 └──┬──┘                   | Service |                 | Repository|
       |                       |                      └────┬────┘                 └────┬──────┘
       |   Authenticate User   |                           |                           |
       +---------------------> |                           |                           |
       |                       |                           |                           |
       |                       |                           |                           |
       |   User Authenticated  |                           |                           |
       | <---------------------+                           |                           |
       |                       |                           |                           |
       |                       |                           |                           |
       +----------------Start New Job Request------------->|                           |
       |                       |                           |                           |
       |                       |                           |                           |
       |                       |                           |   Start Job, add to repo  |
       |                       |                           +-------------------------->|
       |                       |                           |                           |
       |                       |                           |                           |
       |<-------------- Return Command ID -----------------|                           |
       |                       |                           |                           |
       |                       |                           |                           |



```

# The Server Library

The Job Service is intended by be used as the entry point of a standalone Go libraray. The `jobService` struct provides the functions: `Start()`, `Stop()`, `Status()` and `Output()` to manipulate jobs.  


Job Service definitaion


```
type JobService
```
```    
	func (*JobService) Start(userId int, cmd string, args []string, cgroup Cgroup) (UUID, error)
```

Start a new job.  Where userId identifies the user starting the job, cmd is the path to the command to run, args is the list of arguments to pass to the command and cgroup provides setup/teardown of the cgroup the command will run in.

Return the UUID of the created command and error if the command could not be started.
```
    func (*JobService) Status(userId int, cmdId UUID) (Job, error)
```
Status a previously started job.  userId is the ID of the user making the request and cmdId is the UUID of the command to status.

On success a Job struct is returned.  The request will error if the userId in the request does not match the userId of the user who initially ran the `Start()` function. 

```
    func (*JobService) Stop(userId int, cmdId UUID) (Job, error)
```
Stop a previously started job.  userId is the ID of the user making the request and cmdId is the UUID of the command to stop.

On success a Job struct is returned.  The request will error if the userId in the request does not match the userId of the user who initially ran the `Start()` function, or if the command is not currently running.

```
    func (*JobService) Output(userId int, cmdId UUID, output <-chan string) error
```
Retrieve the output of a previously started command.  userId is the ID of the user making the request, cmdID is the UUID of the command to retrieve the output on and output is the channel which will stream the output one line at a time.  The channel will close when all data is read or if the command is stopped prematurely.

```
type Cgroup
```
```
    func (*Cgroup) Setup(mountPoint string,  subDir string) error
```
This function creates a new mountpoint for the `/sys/fs/cgroup` directory and creates a unique sub directory for the job to run in.  

The various values for CPU, memory and I/O will be hard coded.  This is done as a siplification for this project.  A production version would be configurable.

```
	func (*Cgroup) Start(pid int) error
```
Insert the pid of the running command into the pid file to put the process in the cgroup.

``` 
func(*Cgroup) Teardown() error
```

Teardown changes made by the Setup() function: remove the sub directory and unmount `/sys/fs/cgroup`.



























