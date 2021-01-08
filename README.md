# Simple Remote Command Protocol

Simple Remote Command Protocol is a protocol for executing remote commands
over a duplex connection such as TCP or Websockets. The purpose of the protocol
is to forward the execution (signals, stderr/stdout/stdin and exit codes) over
such a network connection.

The protocol can be seen as a much simplified version of ssh which supports
the bare minimum required to forward a remote command, and that expects transport
authentication and encryption to be provided by the underlying transport protocol.

# Specification


Each packet consists of a header, followed by a BARE encoded message.


## Packet Headers

A packet type followed by a size, where size is the amount of data to be read from the connection.

``` type:u8 ++ size:u32/be  ```

## Packets

### Exec

Type=0
Direction=client->server

Intial packet, a request from the client to the server. If command is not specified, the server may 
choose a command.

```
type Exec {
  command: Option<{
    bin: String
    args: []String
  }>
}
```

## AckExec

Type=1
Direction=server->client

Ack packet confirming exec.

```
type AckExec {
  stdin_window_size: uint
  stdout_window_size: uint
  stderr_window_size: uint
  max_data_packet_size: uint
}
```

## NackExec

Type=2
Direction=server->client

Nack packet rejecting exec.


```
type NackExec {
  reason: String
}
```

## WindowAdjust

Type=3
Direction=client<->server

Window adjust packet used for flow control, similar to how ssh channels perform flow control.

```
type WindowAdjust {
  descriptor: (Stdin|Stdout|Stderr)
  amount: uint
}
```

## Data

Type=4
Direction=client<->server

Data packets containing application input/output.

```
type Data {
  descriptor: (Stdin|Stdout|Stderr)
  data: data<$size-1>
}
```

## Close

Type=5
Direction=client<->server

Sent when a file closes.

```
type Close {
  descriptor: (Stdin|Stdout|Stderr)
}
```

Notes:
 - each is split into its own type to save the enum byte.
 - the data is fixed length in terms of the BARE spec, but the actual length is computed based on the packet header.

## Signal

Type=6
Direction=client->server

```
type Signal {
  sig: INT | TERM
}
```

Note - For the current version of the spec, only a subset of signals are defined, it is up to the application
to map these signals to signals appropriate for the OS.

## Exit

Type=7
Direction=server->client

```
type Exit {
  Status: int
}
```

# Description

A client requests an exec from the server, the server replies either AckExec or NackExec. NackExec signals
the end of the connection.

On a reply of AckExec the server and client being to exchange data packets, signal packets and close packets. 

The server is permitted to send an exit packet after both stdout and stderr have been closed.

Stdin/Stderr/Stdout all maintain a data window counter for flow control, starting full, where more data may only be sent if the other end confirms it with a window adjust message.