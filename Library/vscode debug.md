# task.json 配置
```
{
    "tasks": [
        {
            "type": "cppbuild",
            "label": "C/C++: g++ build active file",  // 任务名，在lanuch.json使用此任务名，从而执行此任务
            "command": "/bin/g++",
            "args": [
                "-fdiagnostics-color=always",
                "-g",
                // "${file}",
                "${fileDirname}/*.cpp",   // 编译当前打开的文件所在目录下的所有.cpp文件
                "-o",
                "${fileDirname}/${fileBasenameNoExtension}.out"  // 目标文件名
            ],
            "options": {
                "cwd": "${fileDirname}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "detail": "调试器生成的任务。"
        }
    ],
    "version": "2.0.0"
}

```

# launch
```
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "g++ - Build and debug active file",  // 名字随便取
            "type": "cppdbg",
            "request": "launch",
            // program 指定调试那个可执行文件，需要绝对路径
            // ${fileDirname} 当前打开的文件所在的绝对路径，不包括文件名
            // ${fileBasenameNoExtension} 当前打开的文件的文件名，不包括路径和后缀名
            "program": "${fileDirname}/${fileBasenameNoExtension}.out", 
            "args": [],
            "stopAtEntry": false,
            "cwd": "${fileDirname}", //  cwd 字段来指定应用程序的当前工作目录，访问资源的时候从cwd指定的目录下访问。
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                },
                {
                    "description": "Set Disassembly Flavor to Intel",
                    "text": "-gdb-set disassembly-flavor intel",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "C/C++: g++ build active file",  // 在执行lanuch.json之前要做的任务
            "miDebuggerPath": "/bin/gdb"        // 指定调试工具
        }
    ]
}


```


# 问题
![[Pasted image 20250303223852.png]]

GDB 的 bug，尚未解决。