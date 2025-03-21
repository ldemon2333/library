The `lsof` (List Open Files) command is a powerful utility in Unix-like operating systems that provides information about files opened by processes. Since Unix treats everything as a file—including regular files, directories, devices, and network connections—`lsof` offers a comprehensive view of system activity.

Find Processes Using a Specific File: To identify which processes are using a particular file:
```
lsof /path/to/file
```

List Open Files by a Specific User: To see all files opened by a specific user:
```
lsof -u username
```

List Open Files by a Specific Process: To list files opened by a specific process ID(PID):
```
lsof -p PID
```

List Open Network Connections: To display all open network connections:
```
lsof -i
```

Find Processes Listening on a Specific Port: To identify processes listening on a specific port.
```
lsof -i :80
```

List Open Files in a Directory: To list all open files within a specific directory:
```
lsof +D /path/to/directory
```

**Understanding `lsof` Output:**

The output of `lsof` includes several columns:

- **COMMAND:** The name of the command associated with the process.
- **PID:** The process ID.
- **USER:** The user who owns the process.
- **FD:** The file descriptor number.
- **TYPE:** The type of the file (e.g., regular file, directory, socket).
- **DEVICE:** The device numbers.
- **SIZE/OFF:** The size of the file or the file offset.
- **NODE:** The inode number.
- **NAME:** The name of the file.

![[Pasted image 20250228133522.png]]