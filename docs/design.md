<a name="_9h4p1coakp1j"></a>TRC Design

Teleport Remote Command

# <a name="_g7gm0m4ewxo6"></a>What
The teleport remote command (trc) provides a means to execute remote commands on a Linux server

# <a name="_h1h63640ztqs"></a>Why
trc will allow Teleport support techs to log into customer's Linux servers and execute shell commands to troubleshoot customer issues.

# <a name="_n02bd3awbvj8"></a>Details
trc is split into two executables: trc-agent and trc-client. trc-client runs in the Teleport environment and sends arbitrary shell commands to trc-agent, which is installed on the customers' Linux box. trc-agent is responsible for executing the received commands and stores the results. The agent and client communicate over gRPC and authenticates over mTLS

Resources used by commands run by the agent are restricted using Linux Control Groups v2 (cgroups). However, the commands are run as root with no whitelisting of allowable commands. 

NOTE: A production version of this tool would typically include allow/deny lists or commands run in a chroot environment  to prevent potentially harmful commands from being run. 

# <a name="_isobl31grue1"></a>trc-client usage
trc-client consists of the sub commands start, stop status and output. All the commands except output return JSON. output streams the output of a command

The following options are common to all subcommands:

`  `Options:

`  `-help

`    	`print help

-host string

`    	`remote host to connect to (default "127.0.0.1")

`  `-ident string

`    	`config file with paths to ssl certs and keys (required)

`  `-port int

`    	`remote port to connect to (default 50051)

`  `-ident string

`    	`config file with paths to ssl certs and keys (required)



Where -ident is the path to a JSON file with information about SSL certs and keys:

{

`	`"ca-cert": "<path to self-signed CA certificate>",

`	`"key": "<path to host's key>",

`	`"host-cert": "<path to host cert signed by 'ca-cert'">

}


### <a name="_6ybw908vo9un"></a>Subcommand error output
On error, all subcommands output the following error format:

{

`  `"error": "error message",

`  `"code": "numerical error code"

}


### <a name="_ibnjqdwwhvf0"></a>start subcommand
start starts a new shell command on the remote server.

Usage: trc-client start [-host|-port|-ident] <command> [command arguments].

Output:

{

`  `"id": "<unique UUID>"

}

id: a unique identifier that can be used to stop or status commands

### <a name="_an8sl31hy99k"></a>status subcommand
The status command retrieves information about a previously started command:

Usage: trc-client [options] status <command id>

Get status of a previously started command.

Output:

{

`  `"id": "<command UUID>",

`  `"command": "<command run>",

`  `"args": [ 

`            `"command arg1",

`            `"command arg2"

`  `],

`  `"state": "[running,completed,canceled,error]",

`  `"error": "null|error message",

`  `"exit\_status: "command exit status"

}


### <a name="_vmj8dmfecyrn"></a>output subcommand
The output subcommand returns output of a command. If the command is still running on the remote machine, the output will be live streamed until the command finishes or is stopped by another trc command. If the command is not running, it will exit after all lines have been displayed. Both stdout AND stderr are included in the output.

Usage: trc-client [options] output <command id>

Stream the output of a command


Output:

command output line1

command output line2

command output line3

### <a name="_xwvk9ga52s"></a>stop subcommand
The stop command stops a running command. If the command is not running, or stopping the command fails, an error is returned.

Output:

{

`  `"success": "true"

}

# <a name="_lzxdkro76353"></a>trc-agent usage

The agent only takes a single option:

Options:

`    `- ident string

`        `config file with paths to ssl certs and keys (required)

File format:

{

`     `"ca-cert": "<path to CA cert>",

`     `"agent-key": "<path to host’s key>",

`     `"agent-cert": "<path to host's cert signed by CA cert>",

}


# <a name="_l5xtktkosihk"></a>gRPC Protocol
The protobuf message definitions for sending and receiving commands between the client and agent:

service TrcAgent{

`    `rpc Start(StartRequest) returns (StartResponse);

`    `rpc Output(IdRequest) returns (stream OutputResponse);

`    `rpc Status(IdRequest) returns (StatusResponse);

`    `rpc Stop(IdRequest) returns (StopResponse);

}

message StartRequest {

`    `string command = 1;

}

message StartResponse {

`    `UUID id = 1;

`    `Error error = 2;

}

message IdRequest {

`    `UUID id = 1;

}

message OutputResponse {

`    `string output = 1;

`    `Error error = 2;

}


message StatusResponse {

`    `UUID id = 1;

`    `string cmd = 2;

`    `repeated string args = 3;

`    `string state = 4;

`    `string error = 5;

`    `int32 exit = 6;

`    `Error error = 7;

}

message StopResponse {

`    `UUID id = 1;

`    `Error error = 2;

}

enum State {

`    `RUNNING = 0;

`    `COMPLETE = 1;

`    `STOPPED = 2;

`    `ERROR = 3;

}

message UUID {

`    `string value = 1;

}

message Error {

`    `int32 code = 1;

`    `string message = 2;

}

# <a name="_lmohbtf2asqg"></a>Authentication and Authorization
Authentication will be handled using mutual TLS (mTLS).  Both the agent and the client will generate self signed certs using a common CA. When the connection is established, the client and agent will each verify the other’s authenticity.   Each client user will generate their own cert.  In this way, the agent can distinguish which user executes which commands.  

Authorization will ensure that a client user (as determined by the cert attached to the request) can only get the stop, get the status and stream the output of commands started by that user.  In other words, if user 1 starts a command, user 2 can’t stop, status or get the output of that command. Internally this will be accomplished by pairing the command execution with the public certificate attached to the request.

# <a name="_tr2es7dqywgj"></a>mTLS
Mutual TLS uses a dual exchange of certificates to verify the authenticity of both the host making the gRPC call and the host receiving the request.

This  requires three files on each the agent and the client:

1) The CA certificate file (shared between agent and client)
1) The host’s self signed certificate file, which is unique to the agent and each client user 
1) The host’s private key, which is also unique to the agent and each client user

gRPC uses the CA certificate file to generate a certificate pool.  The self signed certificate and the private key are used to generate the certificate which will be presented in the authentication.

By default, the[ TLS package](https://go.dev/src/crypto/tls/cipher_suites.go) only allows cipher suites with no known vulnerabilities. These are the encryption algorithms that this project will use.


# <a name="_1y2ukim9xwfx"></a>Job Service

The job service is a library responsible for managing jobs. The job service provides the interface to start, stop, status and get output of jobs.
As each job is started, it sets up the cgroup by creating the directory and setting up the appropriate files. It then manages the lifecycle of the job, both notifying the job if the client stops the job and receiving notification as jobs complete. As the job changes state, the job service updates the job repository.

The lifecycle of a job:

1. The cgroup mounts, directories and files are setup
1. The command is executed using exec.cmd Go library. The process is started in a goroutine then blocks waiting for Cmd.Wait() to return. When the command is started, the pid is added to cgroup.procs file and the Cmd object is saved in the JobService. The command is called through a wrapper script which ensures that the PID of the process is added to the procs file before execution begins (see cgroups section below for details)
1. JobService updates that job's state in the repo to running
1. If the process is stopped prematurely via a call to Stop(): the JobService calls the job's Cmd.Process.Kill() method.
1. On process completion, the job's state is updated in the repo
1. The setup done for cgroups is cleaned up.

As each job is executed, the output is written to a file.  For the purposes of this project, STDOUT and STDERR are both written to the same file and are interleaved.  A more robust system would separate the outputs and allow callers to specify which should be returned.

# <a name="_dp6btk223uz"></a>Jobs Repository
The job repository preserves the state of each job. For this implementation it is not backed by a database, but the schema is:

`     `command\_id        |     user\_id        |    state     |   command     |    args       |    output\_file.  | exit        | error

`   `--------------------+--------------------+--------------+---------------+---------------+------------------+-------------+------

`   `UUID of the command | ID of user who     | state of the | command which | list of args  | file with command| command exit| error starting

`   `which was run.      | started command.   | command.     | was run.      | to the command| output.          | status code | command














