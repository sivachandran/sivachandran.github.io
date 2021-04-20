---
layout: post
title: "Troubleshooting .NET 'module could not be found' exception with ProcessMonitor"
date: 2013-03-17 14:25:00 +0530
tags: [dotnet, win32]
---

In this post I am going to share a technique to troubleshoot "module could not be found" exception in managed/.NET application due to missing native DLL dependency. The difficulty in troubleshooting this exception is it doesn't provide any information about the missing DLL(e.g. file name) apart from the exception message "module could not be found".

### The Problem
To demonstrate the problem I am going to create an managed/.NET console app which depends on C++/CLI DLL which in-turn depends on a native Win32 DLL. The dependency looks like below

![Dependency](/img/cli-app-dependency.jpg)

and the implementation of executable and DLLs are

```c#
// ManagedApp.cs
namespace ManagedApp
{
    class Program
    {
        static void Main(string[] args)
        {
            CppCliDll.CppCliClass.Print();
        }
    }
}
```

```c++
// CppCliDll.h
#pragma once

#include <NativeDll/NativeClass.h>

namespace CppCliDll
{
    public ref class CppCliClass
    {
    public:
        static void Print()
        {
            NativeDll::NativeClass::Print();
        }
    };
}
```

```c++
// NativeDll.h
#ifdef NATIVEDLL_EXPORTS
#define NATIVEDLL_API __declspec(dllexport)
#else
#define NATIVEDLL_API __declspec(dllimport)
#endif

#include <stdio.h>

namespace NativeDll
{
    // This class is exported from the NativeDll.dll
    class NATIVEDLL_API NativeClass
    {
    public:
        static void Print()
        {
            printf("Hello from native dll");
        }
    };
}
```

for readability the DLLs are implemented in the header itself.

So with this dependency and implementation if we place all the three binaries(ManagedApp.exe, CppCliDll.dll and NativeDll.dll) in same directory and run the ManagedApp.exe we will get the following expected output.

```cmd
C:\Demo> ManagedApp.exe
Hello from native dll
```

whereas if the NativeDll.dll is not present in the directory then we will get the "module could not be found" exception

```cmd
C:\Demo> del NativeDll.dll
C:\Demo> ManagedApp.exe
Unhandled Exception: System.IO.FileNotFoundException: The specified module could not be found. (Exception from HRESULT: 0x8007007E)
   at ManagedApp.Program.Main(String[] args)
```

the exception is exactly same even if we run the ManagedApp.exe in Visual Studio debug mode

![Debug Screenshot 1](/img/debug-screenshot.jpg)

the exception doesn't provide any clue about which DLL is missing. If the application depends on small no. of DLLs(like the ManagedApp.exe) then it is relatively easy to guess which DLL could be missing. But it will be really difficult to guess if the application depends(directly and indirectly) on more no. of (e.g. 20+) DLLs. It will be even more difficult if one of our dependency DLL assumes presence of system wide DLL like libcrypto.dll(part of OpenSSL).

The strange thing here is if the managed DLL(here CppCliDll.dll) is not present in the directory then we'll get a exception with information explaining exactly which DLL is missing.

```cmd
C:\Demo> del CppCliDll.dll
C:\Demo> ManagedApp.exe
Unhandled Exception: System.IO.FileNotFoundException: Could not load file or assembly 'CppCliDll, Version=1.0.4824.22063, Culture=neutral, PublicKeyToken=null' or one of its dependencies. The system cannot find the file specified.
File name: 'CppCliDll, Version=1.0.4824.22063, Culture=neutral, PublicKeyToken=null'
   at ManagedApp.Program.Main(String[] args)
...
```

But I don't understand why that crucial information is not available in the exception when the native dependency of managed app/DLL is missing. I also tried the Fusion logs but even that didn't provide any clue which DLL we are missing. I've wasted many hours trying to find the missing DLL before finding the following technique.

### The Technique
The technique I finally found to troubleshoot this exception is using the ProcessMonitor. ProcessMonitor is a tool provided by Microsoft(originally SysInternals which is bought by Microsoft) and it can monitor system events like access to file, network, registry and etc. It provides various options like filtering in which it will show events related to specific process.

So to find out the missing DLL start the ProcessMonitor with the following capture options

- Enable "Show File System Activity" and disable everything else
- Add filter for "Process Name" field containing with value as our application name(it is ManagedApp for the demo app)
- Add highlighting for "Path" field ending with value "dll"
- Add highlighting for "Result" field containing with value "NOT FOUND"

![Debug Screenshot 2](/img/debug-screenshot-2.jpg)
![Debug Screenshot 3](/img/debug-screenshot-3.jpg)

now run the application and it will show the same error message that we've observed earlier. Now stop capturing events in the ProcessMonitor and look for for highlighted row starting from the latest event(bottom) to oldest event(top). When you find a highlighted row observe the corresponding DLL file name and make sure the same DLL file name is not opened successfully in the future(towards bottom) or past(towards top). You can also narrow down the search space by considering file opening attempt that happened only in the application directory(the directory where ManagedApp.exe present).

![Debug Screenshot 4](/img/debug-screenshot-4.jpg)

in the above screenshot the red highlighted row shows that an attempt to open "NativeDll.dll" failed and the loader tries to open the DLL in various other places like "C:\Windows\system" and etc. If you fully analyze the events you can confirm that all attempts to open the DLL would have failed. So this clearly shows that the missing DLL is "NativeDll.dll" file.

This approach of analyzing the ProcessMonitor file system events is little tricky and as well as time consuming. That is why I came up with the following python script which can analyze the ProcessMonitor file system events exported as CSV and it can report the missing DLLs.

```cmd
C:\Demo> python FindMissingDll.py Logfile.CSV
Attemmpt to open the following dlls failed:
mscorrc.dll.dll
rpcss.dll
nativedll.dll
wow64log.dll
```

As you can see, it also reported failure attempts to open other DLLs apart from NativeDll.dll. These are all false-positives, failure to open these DLLs will not affect the executability of the application. The source of the python script is

```python
import sys
import os

if __name__ == '__main__':
    if len(sys.argv) != 2:
        print "Usage: FindMissingDll <process monitor csv exported file>"
        sys.exit(0)
        
    csv_lines = open(sys.argv[1]).readlines()
    
    # the following dictionary contains the status of all dlls. the key for the dictionary
    # is the dll file name(lower case) and value is the status of whether that dll successfully
    # opened once(an attempt to open the same file in some other path may fail which we shouldn't
    # consider as missing dll)
    dll_status = {}
    
    for line in csv_lines:
        fields = line.replace("\"", "").split(',')
        
        # process monitor CSV file fields are
        # "Time of Day","Process Name","PID","Operation","Path","Result","Detail"
        operation = fields[3]
        result = fields[5]
        if operation == "IRP_MJ_CREATE":
            path = fields[4]
            file_name = os.path.basename(path).lower()
            
            if not file_name.endswith(".dll"):
                continue
            
            if result == "SUCCESS":
                dll_status[file_name] = True
            elif result == "NAME NOT FOUND" and file_name not in dll_status:
                dll_status[file_name] = False

    if len(dll_status) > 0:
        print "Attemmpt to open the following dlls failed:"
        for file_name in dll_status:
            if not dll_status[file_name]:
                print file_name
```

So with this technique and the script I hope whoever struggling with "module could not be found" exception can easily find the missing DLL(s).
