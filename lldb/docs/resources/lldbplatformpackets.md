# LLDB Platform Packets

Here is a brief overview of the packets that an lldb platform server
needs to implement for the lldb testsuite to be run on a remote
target device/system.

These are almost all lldb extensions to the gdb-remote serial
protocol.  Many of the `vFile:` packets are also described in the "Host
I/O Packets" detailed in the gdb-remote protocol documentation,
although the lldb platform extensions include packets that are not
defined there (`vFile:size:`, `vFile:mode:`, `vFile:symlink`, `vFile:chmod:`).

Most importantly, the flags that lldb passes to `vFile:open:` are
incompatible with the flags that gdb specifies.

## QStartNoAckMode

### Brief

A request to stop sending ACK packets for each properly formatted packet.

### Example

A platform session will typically start like this:
```
receive: +$QStartNoAckMode#b0
send:    +       <-- ACKing the properly formatted QStartNoAckMode packet
send:    $OK#9a
receive: +       <-- Our OK packet getting ACKed
```

ACK mode is now disabled.

## qHostInfo

### Brief

Describe the hardware and OS of the target system

### Example

```
receive: qHostInfo
send:    cputype:16777228;cpusubtype:1;ostype:ios;watchpoint_exceptions_received:before;os_version:12.1;vendor:apple;default_packet_timeout:5;
```

All numbers are base 10, `os_version` is a string that will be parsed as major.minor.patch.

## qModuleInfo

### Brief

Get information for a module by given module path and architecture.

The response is:
* `(uuid|md5):...;triple:...;file_offset:...;file_size...;` or
* `EXX` - for any errors

### Example

```
receive: qModuleInfo:2f62696e2f6c73;
```

## qGetWorkingDir

### Brief

Get the current working directory of the platform stub in
ASCII hex encoding.

### Example

```
receive: qGetWorkingDir
send:    2f4170706c65496e7465726e616c2f6c6c64622f73657474696e67732f342f5465737453657474696e67732e746573745f646973617373656d626c65725f73657474696e6773
```

## QSetWorkingDir

### Brief

Set the current working directory of the platform stub in
ASCII hex encoding.

### Example

```
receive: QSetWorkingDir:2f4170706c65496e7465726e616c2f6c6c64622f73657474696e67732f342f5465737453657474696e67732e746573745f646973617373656d626c65725f73657474696e6773
send:    OK
```

## qPlatform_mkdir

### Brief

Create a directory on the target system.

### Example

```
receive: qPlatform_mkdir:000001fd,2f746d702f6131
send:    F0
```

request packet has the fields:
   1. mode bits in base 16
   2. file path in ASCII hex encoding

response is F followed by the return value of the `mkdir()` call,
base 16 encoded.

## qPlatform_shell

### Brief

Run a shell command on the target system, return the output.

### Example

```
receive: qPlatform_shell:6c73202f746d702f,0000000a
send:    F,0,0,<OUTPUT>
```

request packet has the fields:
   1. shell command in ASCII hex encoding
   2. timeout
   3. working directory in ASCII hex encoding (optional)

Response is `F` followed by the return value of the command (base 16),
followed by another number, followed by the output of the command
in binary-escaped-data encoding.

## qLaunchGDBServer

### Brief

Start a gdbserver process (`gdbserver`, `debugserver`, `lldb-server`)
on the target system.

### Example

```
receive: qLaunchGDBServer;host:<HOSTNAME_LLDB_IS_ON>;
send:    pid:1337;port:43001;
```

Request packet hostname field is not ASCII hex encoded. Hostnames
do not have `$` or `#` characters in them.

Response to the packet is the pid of the newly launched gdbserver,
and the port it is listening for a connection on.

When the testsuite is running, lldb may use the pid to kill off a
debugserver that doesn't seem to be responding, etc.

## qKillSpawnedProcess

### Brief

Kill a process running on the target system.

### Example

```
receive: qKillSpawnedProcess:1337
send:    OK
```
The request packet has the process ID in base 10.

## qProcessInfoPID:

### Brief

Gather information about a process running on the target.

### Example

```
receive: qProcessInfoPID:71964
send:    pid:71964;name:612e6f7574;
```

The request packet has the pid encoded in base 10.

The reply has semicolon-separated `name:value` fields, two are
shown here. `pid` is base 10 encoded. `name` is ascii hex encoded.
lldb-server can reply with many additional fields, but this is probably
enough for the testsuite.

## qfProcessInfo

### Brief

Search the process table for processes matching criteria,
respond with them in multiple packets.

### Example

```
receive: qfProcessInfo:name_match:equals;name:6e6f70726f6365737365786973747377697468746869736e616d65;
send:    pid:3500;name:612e6f7574;
```

The request packet has a criteria to search for, followed by
a specific name.

| Key          | Value     | Description
| ------------ | --------- | -----------
| `name`       | ascii-hex | An ASCII hex string that contains the name of the process that will be matched.
| `name_match` | enum      | One of: `equals`, `starts_with`, `ends_with`, `contains` or `regex`
| `pid`        | integer   | A string value containing the decimal process ID
| `parent_pid` | integer   | A string value containing the decimal parent process ID
| `uid`        | integer   | A string value containing the decimal user ID
| `gid`        | integer   | A string value containing the decimal group ID
| `euid`       | integer   | A string value containing the decimal effective user ID
| `egid`       | integer   | A string value containing the decimal effective group ID
| `all_users`  | bool      | A boolean value that specifies if processes should be listed for all users, not just the user that the platform is running as
| `triple`     | ascii-hex | An ASCII hex target triple string (`x86_64`, `x86_64-apple-macosx`, `armv7-apple-ios`)

If no criteria is given, `qfProcessInfo` will request a list of every process.

The lldb testsuite currently only uses `name_match:equals` and the
no-criteria mode to list every process.

The response should include any information about the process that
can be retrieved in semicolon-separated `name:value` fields.
In this example, `pid` is base 10, `name` is ASCII hex encoded.
The testsuite seems to only require these two.

This packet only responds with one process. To get further matches to
the search, `qsProcessInfo` should be sent.

If no process match is found, `Exx` should be returned.

Sample packet/response:
```
send packet: $qfProcessInfo#00
read packet: $pid:60001;ppid:59948;uid:7746;gid:11;euid:7746;egid:11;name:6c6c6462;triple:7838365f36342d6170706c652d6d61636f7378;#00
send packet: $qsProcessInfo#00
read packet: $pid:59992;ppid:192;uid:7746;gid:11;euid:7746;egid:11;name:6d64776f726b6572;triple:7838365f36342d6170706c652d6d61636f7378;#00
send packet: $qsProcessInfo#00
read packet: $E04#00
```

## qsProcessInfo

### Brief

Return the next process info found by the most recent `qfProcessInfo:`
packet.

### Example

Continues to return the results of the `qfProcessInfo`. Once all matches
have been sent, `Exx` is returned to indicate end of matches.

## qPathComplete

### Brief

Get a list of matched disk files/directories by passing a boolean flag
and a partial path.

### Example

```
receive: qPathComplete:0,6d61696e
send:    M6d61696e2e637070
receive: qPathComplete:1,746573
send:    M746573742f,74657374732f
```

If the first argument is zero, the result should contain all
files (including directories) starting with the given path. If the
argument is one, the result should contain only directories.

The result should be a comma-separated list of hex-encoded paths.
Paths denoting a directory should end with a directory separator (`/` or `\`).

## vFile:size

### Brief

Get the size of a file on the target system, filename in ASCII hex encoding.

### Example

```
receive: vFile:size:2f746d702f61
send:    Fc008
```

response is `F` followed by the file size in base 16.
`F-1,errno` with the errno if an error occurs, base 16.

## vFile:mode

### Brief

Get the mode bits of a file on the target system, filename in ASCII hex.

### Example

```
receive: vFile:mode:2f746d702f61
send:    F1ed
```

response is `F` followed by the mode bits in base 16, this `0x1ed` would
correspond to `0755` in octal.
`F-1,errno` with the errno if an error occurs, base 16.

## vFile:unlink

### Brief

Remove a file on the target system.

### Example

```
receive: vFile:unlink:2f746d702f61
send:    F0
```

Argument is a file path in ascii-hex encoding.

Response is `F` plus the return value of `unlink()` in base 16 encoding.
If unlink failed, the return value may be followed by a comma and the value of
errno in base 16 encoding.

## vFile:symlink

### Brief

Create a symbolic link (symlink, soft-link) on the target system.

### Example

```
receive: vFile:symlink:<SRC-FILE>,<DST-NAME>
send:    F0,0
```

Argument file paths are in ascii-hex encoding.
Response is `F` plus the return value of `symlink()`, base 16 encoding,
optionally followed by the value of errno if it failed, also base 16.

## vFile:chmod / qPlatform_chmod

### Brief

Change the permission mode bits on a file on the target

### Example

```
receive: vFile:chmod:180,2f746d702f61
send:    F0
```

Arguments are the mode bits to set, base 16, and a file path in
ascii-hex encoding.
Response is `F` plus the return value of `chmod()`, base 16 encoding.

These 2 packets do the same thing, it is not known why we ended up with 2.

## vFile:chmod

### Brief

Change the permission mode bits on a file on the target.

### Example

```
receive: vFile:chmod:180,2f746d702f61
send:    F0
```

Arguments are the mode bits to set, base 16, and a file path in
ascii-hex encoding.
Response is `F` plus the return value of `chmod()`, base 10 encoding.

## vFile:open

### Brief

Open a file on the remote system and return the file descriptor of it.

### Example

```
receive: vFile:open:2f746d702f61,00000001,00000180
send:    F8
```

request packet has the fields:
   1. ASCII hex encoded filename
   2. Flags passed to the open call, base 16.
      Note that these are not the `oflags` that `open(2)` takes, but
      are the constant values in `enum OpenOptions` from LLDB's
      [`File.h`](https://github.com/llvm/llvm-project/blob/main/lldb/include/lldb/Host/File.h).
   3. Mode bits, base 16

response is `F` followed by the opened file descriptor in base 16.
`F-1,errno` with the errno if an error occurs, base 16.

## vFile:close

### Brief

Close a previously opened file descriptor.

### Example

```
receive: vFile:close:7
send:    F0
```

File descriptor is in base 16. `F-1,errno` with the errno if an error occurs,
errno is base 16.

## vFile:pread

### Brief

Read data from an opened file descriptor.

### Example

```
receive: vFile:pread:7,1024,0
send:    F4;a'b\00
```

Request packet has the fields:
   1. File descriptor, base 16
   2. Number of bytes to be read, base 16
   3. Offset into file to start from, base 16

Response is `F`, followed by the number of bytes read (base 16 encoded), a
semicolon, followed by the data in the binary-escaped-data encoding.

## vFile:pwrite

### Brief

Write data to a previously opened file descriptor.

### Example

```
receive: vFile:pwrite:8,0,\cf\fa\ed\fe\0c\00\00
send:    F1024
```

Request packet has the fields:
   1. File descriptor, base 16
   2. Offset into file to start from, base 16
   3. binary-escaped-data to be written

Response is `F`, followed by the number of bytes written (base 16 encoded).

## Launching Processes

Finally, the platform must be able to launch processes so that debugserver
can attach to them. To do this, the following packets should be handled:
* `QSetDisableASLR`
* `QSetDetachOnError`
* `QSetSTDOUT`
* `QSetSTDERR`
* `QSetSTDIN`
* `QEnvironment`
* `QEnvironmentHexEncoded`
* `A`
* `qLaunchSuccess`
* `qProcessInfo`

Most of these are documented in the standard gdb-remote protocol
and/or LLDB's [GDB Remote Protocol Extensions](lldbgdbremote).
