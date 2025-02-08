Remote Procedure Call (RPC) is a power technique for constructing 分布式，client-server based applications.

The called procedure need not exist in the same address space as the calling procedure. The two processes may be on the same system, or they may be on different systems with a network connnection them.

# How to make a Remote Procedure Call
![[Pasted image 20250108153817.png]]
# Types of RPC
## Callback RPC
Callback RPC allows processes to act as both clients and servers. It helps with remote processing of interactive applications. The server gets a handle to the client, and the client waits during the callback. This type of RPC manages callback deadlocks and enables peer-to-peer communication between processes.

## Broadcast RPC
In Broadcast RPC, a client’s request is sent to all servers on the network that can handle it. Broadcast RPC helps reduce the load on the network.

## Batch-mode RPC
Batch-mode RPC collects multiple RPC requests on the client side and sends them to the server in one batch. This reduces the overhead of sending many separate requests. Batch-mode RPC works best for applications that don’t need to make calls very often. It requires a reliable way to send data.

# What does RPC do?
When a program with RPC is make ready to run, it includes a helper part called a stub. This stub acts like the remote code. When the program runs and tries to use the remote code, the stub gets this request. It then sends it to another helper program on the same computer. The first time this happens, the helper program asks a special computer where to find the remote code.

# Working of a RPC
![[Pasted image 20250108155201.png]]
1. A client invokes a client stub procedure, passing parameters in the usual way. The client stub resides within the client's own address space.
2. The client stub pack the parameters into a message. Marshalling includes converting the representation of the parameters into a standard format, and copying each parameter into the message.
3. The client stub passes the message to the transport layer, which sends it to the remote server machine.  On the server, the transport layer passes the message to a server stub, which demarshalls(unpack) the parameters and calls the desired server routine using the regular procedure call mechanism.
4. When the server procedure completes, it returns to the server stub (e.g., via a normal procedure call return), which marshalls the return values into a message.
5. The server stub then hands the message to the transport layer. The transport layer sends the result message back to the client transport layer, which hands the message back to the client stub.
6. The client stub demarshalls the return parameters and execution returns to the caller.

# Issues of the RPC
## RPC Runtime
RPC run-time system is a library of routines and a set of services that handle the network communications that underlie the RPC mechanism. In the course of an RPC call, client-side and server-side run-time systems’ code handle binding, establish communications over an appropriate protocol, pass call data between the client and server, and handle communications errors.

## Stub
The function of the stub is to provide transparency to the programmer-written application code. On the client side, the stub handles the interface between the client’s local procedure call and the run-time system, marshalling and unmarshalling data, invoking the RPC run-time protocol, and if requested, carrying out some of the binding steps. 

On the server side, the stub provides a similar interface between the run-time system and the local manager procedures that are executed by the server.

## Binding
The most flexible solution is to use dynamic binding and find the server at run time when the RPC is first made.

## The call semantics associated with RPC
