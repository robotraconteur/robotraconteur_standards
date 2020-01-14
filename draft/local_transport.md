<p align="center"><img src="../images/RRheader2.jpg"></p>

# Robot Raconteur Value Types Standard

http://robotraconteur.com

http://github.com/robotraconteur

Copyright &copy; 2020 Wason Technology, LLC

*Robot Raconteur is a communication framework designed for use with robotics, automation, building control, and internet of things applications.*

## Abstract

The Local Transport is used to communicate between processes on the same machine. It uses UNIX sockets for communication, and files for discovery. The Local Transport can function as both a client and a server.

## Introduction

The Local Transport is used when nodes running on the same machine want to communicate between each other. The tranport uses UNIX sockets for communication, and files for discovery. UNIX sockets are widely supported on operating systems, including Linux, Mac OSX, and QNX. Until recently, Windows did not support UNIX sockets. Instead, Named Pipes had to be used. UNIX sockets were made available in Windows 10 update 1703. Robot Raconteur version 0.9.2 and greater now use UNIX sockets for Windows as well. If UNIX sockets are not available, the Local Transport is automatically disabled.

## Local Transport URLs

Local Transport servers are identified by some combination of NodeID and/or NodeName. If the desired node is run by the same user, the following URL format is used (See [Robot Raconteur Framework Architecture Standard](framework_architecture.md)):

    rr+local:///?nodeid=<nodeid>&nodename=<nodename>&service=<servicename>

For the Local Transport, the NodeID, NodeName, or both must be specified. If the desired node is run by a different user, the following URL format can be used:

    rr+local://<user>@localhost/?nodeid=<nodeid>&nodename=<nodename>&service=<servicename>

In most cases, the <user> in this URL will be `LocalService` on Windows and `root` on non-windows to access a system service, or the <user> will be the current user. It is recommended to always include the <user> when on an untrusted machine.

## NodeID Caching

When used as a server, the Local Transport may either be started by NodeID, or by NodeName. When started by NodeID, the node is not assigned a name. When started by NodeName, the Local Transport will retrieve the cached NodeID corresponding to the name from files on disk. The files are stored in the following locations:

| Operating System | User NodeName Config Directory |
| ---  | ---  |
| Linux, OSX, and others | $HOME/.config/RobotRaconteur/nodeids |
| Windows | <CSIDL_LOCAL_APPDATA>\RobotRaconteur\nodeids |

*The directories for CSIDL_LOCAL_APPDATA are retrieved using the SHGetFolderPath Win32 command.*

The files have he same name a the NodeName, and contain the NodeID in bracketed 8-4-4-4-12 format `{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}`.

If no file is found for the specified NodeName, once should be generated with a random NodeID.

When in use, the NodeName file must be locked. This is accomplished using the following command to open the file for writing on Windows:

    CreateFile(<path>, GENERIC_READ | GENERIC_WRITE, FILE_SHARE_READ, NULL, OPEN_ALWAYS, FILE_ATTRIBUTE_NORMAL | FILE_FLAG_DELETE_ON_CLOSE, NULL)

or locked using the following command on Linux/OSX:

    fd1 = open(<path>, O_CLOEXEC | O_RDWR | O_APPEND | O_CREAT, S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP);
    struct ::flock lock;
    lock.l_type = F_WRLCK;
    lock.l_whence = SEEK_SET;
    lock.l_start = 0;
    lock.l_len = 0;
    fcntl(fd1, F_SETLK, &lock)

If the locking operating fails (on either Windows or other), it must be assumed that the NodeName is already in use. The attempt to open the server must be aborted.

**Note that this method of caching NodeIDs is used to assign the NodeID when using `ServerNodeSetup` in the core library.**

## Runtime Filesystem Directories and Files

Operating systems have the concept of "runtime" filesystem directories. These directories hold ephemeral data that is not persisted over time. Robot Raconteur uses these runtime directories to store UNIX sockets and the discovery files.

When running as a server, the Local Transport server can be configured to be "public" or "private". When running as "private", the UNIX sockets and discovery files are stored in a location that is only accessible by the same user. Run running as "public", the UNIX sockets and discovery files are stored in a location that can be accessed by all users belonging to the "robotraconteur" group.

The root filesystem directories for Robot Raconteur are as follows:

| Operating System | Normal User Private Directory | Root User Private Directory | Public Directory |
| ---              | ---                    | ---                   | --- |
| Linux            | $XDG_RUNTIME_DIR/robotraconteur | /var/run/robotraconteur/root | /var/run/robotraconteur/$USER |
| Windows          | <CSIDL_LOCAL_APPDATA>\RobotRaconteur\run | <CSIDL_LOCAL_APPDATA>\RobotRaconteur | <CSIDL_COMMON_APPDATA>\RobotRaconteur\%USERNAME% |
| Mac OSX          | $TMPDIR/../C/robotraconteur | /var/run/robotraconteur/root | /var/run/robotraconteur/$USER |

*$XDG_RUNTIME_DIR is created by Systemd. It is typically a value like /var/run/user/1000, where 1000 is the UID of the user. The environmental variable is used.*
*The directories for CSIDL_LOCAL_APPDATA and CSIDL_COMMON_APPDATA are retrieved using the SHGetFolderPath Win32 command.*
*The private user directories for Mac OSX are stored in the per user cache directories under /var/folders. The location is retrieved using the $TMPDIR environmental variable, moving to the parent directory, and moved to the cache directory named "C".

It may be necessary to create the specified directories if they do not exist. If created, these directories must have the appropriate permissions for their use. Private directories should only be accessible by the specified user, and public directories must be accessible by the "robotraconteur" group.

For the remainder of this section <RR_RUNDIR> refers to any of the directories above.

### `socket` directory

The UNIX socket address structure, `sockaddr_un`, has a path limit of 107 or less characters. This is an immediate problem on modern filesystems, when paths can very easily exceed this length. For this reason, a special socket directory is used to store all sockets opened by Robot Raconteur. It has the following location:

    <RR_RUNDIR>/socket/

Sockets in the directory should be named with 16 random letters and numbers, with an extensions `.sock` . For example, a socket may have the following path:

    <RR_RUNDIR>/socket/jABy7UNc04GPrPJZ.sock

Sockets should be deleted when closed.

### by-nodeid and by-nodename

Discovery files are stored in the by-nodeid and by-nodename directories.

    <RR_RUNDIR>/transport/local/by-nodeid/
    <RR_RUNDIR>/transport/local/by-nodename/

The files in these directories contain info files about the Local Transport servers that may be connected by clients, and PID files for locking. 

PID files contain the program ID of the server node, stored as a plain text decimal number.

The info file is stored in plain text, with a name and value pair on each line. The key and value are separated by a colon. The contents of the file are as follows:

| Name | Value Description |
| --   | --- |
| pid | The process ID of the program running the service as a decimal string |
| username | The username running the server process as a string |
| nodename | The NodeName as a string |
| nodeid | The NodeID as a 8-4-4-4-16 bracketed UUID string |
| socket | The file path to the socket accepting UNIX socket connections |
| ServiceStateNonce | The nonce of the service state |

An info and PID file is expected to be saved in the `by-nodeid` and `by-nodename` folders. The `by-nodeid` files should be saved with the filename composed of the bracketed 8-4-4-4-16 UUID, with the extension `.pid` for the PID file, and `.info` for the info file. The `by-nodename` filenames should be composed of the NodeName string, with the extension `.pid` for the PID file, and `.info` for the info file.

When in use, the info and PID files should be locked. This is accomplished using the following command to open the file for writing on Windows:

    CreateFile(<path>, GENERIC_READ | GENERIC_WRITE, FILE_SHARE_READ, NULL, OPEN_ALWAYS, FILE_ATTRIBUTE_NORMAL | FILE_FLAG_DELETE_ON_CLOSE, NULL)

or locked using the following command on Linux/OSX:

    fd1 = open(<path>, O_CLOEXEC | O_RDWR | O_APPEND | O_CREAT, S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP);
    struct ::flock lock;
    lock.l_type = F_WRLCK;
    lock.l_whence = SEEK_SET;
    lock.l_start = 0;
    lock.l_len = 0;
    fcntl(fd1, F_SETLK, &lock)

If the locking operating fails (on either Windows or other), it must be assumed that the NodeName and/or NodeID are already in use. The attempt to open the server must be aborted.

Info and PID files should always be created in the private user directory. If the Local Transport server is opened in public mode, the info and PID files should also be created in the user's public directory.

## Finding Service Nodes

The directories specified in the previous section are searched to find info files describing available service nodes. When a user specifies a URL, these directories should be interrogated to find the correct socket based on NodeID, NodeName, and user. When no user is specified, the current users public, the current users private, and system public directories should be searched. When a username is specified, the specified user's public directory should be searched. If the specified current user's public directory is also specified, the current user's private directory should also be searched.

When interrogating info files, the files must be open read-only avoiding the lock currently held by the Local Transport server. On Windows, the following command is used to open the files:

    CreateFile(<path>, GENERIC_READ, FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL)

on Linux, OSX, and others, use:

    open(<path>, O_CLOEXEC | O_RDONLY)

Once the correct info file has been found, the path to the socket can be extracted. Use the socket `connect()` command with a `sockaddr_un` address to connect.

## Normal Operation

The Local Transport uses the basic stream protocol, as discussed in [Robot Raconteur Basic Stream Protocol Standard](basic_stream_protocol.md), over UNIX sockets. Because UNIX sockets are perfectly reliable while programs are running normally, there is little need for error correction. UNIX sockets use the Berkeley sockets API, similar to TCP sockets. The "bind", "listen", and "accept" commands are used to accept incoming connection. The UNIX sockets are bound to a file within the <RR_RUNDIR>/socket/ directory as discussed previously. Clients use the info files contained in <RR_RUNDIR> directories to find the socket path for the specified node. The Berkeley socket "connect" command is used to establish a connection. Once connected, "send" and "recv" are used to transmit the messages using the basic stream protocol. The server should close the socket connection after transmitting a  DisconnectClientRet (110) message.

## Node Discovery

Node discovery uses Node Announce packets to detect other nodes, as discussed in [Robot Raconteur Node and Service Discovery Standard](node_discovery.md). For the Local Transport, the information to assemble Node Announce packets is stored in the info files, as described previously in this document. The Local Transport running on each process must emulate the behavior of node announce packets, either by periodically generating packets, or using a different method to maintain the cache of detected nodes. Changes in the available nodes or node states should be accomplished using file system event monitoring.
