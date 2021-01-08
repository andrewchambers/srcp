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

``` TYPE:u8 ++ SIZE:u32/be  ```

## Packets

### Exec

TYPE=0

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

TYPE=1

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

TYPE=2

Nack packet rejecting exec.

```
type NackExec {
  reason: String
}
```

## WindowAdjust

TYPE=3

Window adjust packet used for flow control, similar to how ssh channels perform flow control.

```
type WindowAdjust {
  descriptor: (Stdin|Stdout|Stderr)
  amount: uint
}
```

## Data

TYPE=4

Data packets containing application input/output.

```
type Data {
  descriptor: (Stdin|Stdout|Stderr)
  data: data<$remaining>
}
```
## Close

TYPE=5

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

TYPE=10

```
type Signal {
  sig: INT | TERM
}
```

Note - For the current version of the spec, only a subset of signals are defined, it is up to the application
to map these signals to signals appropriate for the OS.

## Exit

Type=11

```
type Exit {
  Status: int
}
```

