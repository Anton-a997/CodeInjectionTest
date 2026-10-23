# CodeInjectionTest
Testing of an executable code injection technique into the address space of Windows OS processes

**This program is developed solely for research and demonstration purposes!**

The solution includes two projects:
1. RemoteShellDll - a dynamic-link library implementing remote shell functionality.
2. AgentInstaller - a loader program for injecting the aforementioned DLL into the address space of another process.

This implementation demonstrates executable code injection as a technique for installing a remote shell (as a DLL) into the address space of another process, as well as methods for controlling a remote computer using it. The method was originally tested on Windows 7, but was also verified to work on Windows 10. It is implemented only for 32-bit processes (on both 32-bit and 64-bit versions of Windows).

## Code Injection Method
Code injection techniques can be divided into two main categories: DLL-based and DLL-less (without a DLL). Using a DLL significantly simplifies the development of the code executed remotely, but serves as a telltale sign during an attack. Injecting machine code directly into the address space is less conspicuous; however, preparing such code is very labor-intensive as it requires writing position-independent code that executes correctly at an a priori unknown memory address and with unknown load addresses of system DLLs. This issue is especially acute in Windows 7 and newer due to ASLR (Address Space Layout Randomization).

This implementation combines both approaches. The remote agent's payload is implemented in a DLL, while the DLL itself is injected into the process address space by injecting position-independent machine code that loads the DLL via API calls.

The position-independent code responsible for loading the DLL must meet the following requirements:
- No absolute, hardcoded memory addresses in the code;
- All function calls are performed via pointers configured immediately prior to injection;
- All data used by the code is stored in a dynamically allocated memory region within a structure loaded alongside the code.

In this implementation, the target process's main thread context hijacking method is used instead of the more common approach of creating a remote thread. At the time of development and testing, this technique allowed avoiding detection by proactive defense modules of antivirus software. The second obstacle when implementing code injection was DEP (Data Execution Prevention). This OS mechanism, introduced in Windows Vista and Windows 7, prevented execution of code placed in dynamically allocated memory regions. Here, this problem is resolved by invoking the code via a RETN instruction, having pushed the return address onto the top of the stack beforehand. The address of the injected code is used as the return address. This approach bypasses DEP execution restrictions in arbitrary regions.

As a result of executing the injected code, the prepared DLL containing the software agent is loaded into the target process address space, and control is returned to the suspended thread at the exact point where it was interrupted.

## Demonstration
To demonstrate the injection process, a console application named AgentInstaller was developed, supporting DLL injection into other processes using the technique described. Internet Explorer (iexplore.exe) was selected as the target process. Both IE and AgentInstaller are running on the same virtual machine.

To inject the library, run the program and enter the target Process ID (PID) into whose context it should be loaded.

![image](https://user-images.githubusercontent.com/88583217/204553287-cbc0b012-b4d3-4fb5-9b73-0821a6cbf979.png)

In the list of process modules, you can see the loaded library named 1.tmp.

![image](https://user-images.githubusercontent.com/88583217/204553572-86cf6f09-8089-4cdb-af9a-63c707c1188a.png)

At this point, the injection stage of the software agent is complete.

The loaded DLL hosts a TCP server that begins listening on port 12321 in a separate thread. This thread is spawned upon DLL loading inside the DllMain() function. A simple TCP client is implemented as a Python 3 script. The client sends user-entered commands to the server and receives the output after execution. On the remote computer, all commands are executed in the context of the Internet Explorer process.

```python
import socket

sock = socket.socket()
ip = input('Remote IP: ')
sock.connect((ip, 12321))

while 1:
    s = input('>>>')
    sock.send(bytearray(s, 'cp1251'))
    s = sock.recv(10240)
    print(s.decode('cp1251'))

sock.close()
```

Capturing cmd output is implemented by redirecting console output to a file. Once the command finishes executing, the agent reads the output file and sends its contents to the client. If there is no output, the agent returns the exit code with which cmd terminated.

### Example: Creating a new user on the remote computer

![image](https://user-images.githubusercontent.com/88583217/204555150-55156442-e828-42f2-bb25-4a110df1ecbe.png)

A new user appears in the Control Panel of the remote machine.

![image](https://user-images.githubusercontent.com/88583217/204555366-7d2ddb82-a777-422d-9a44-d5eca3308ada.png)

This user can be added to the Administrators group.

![image](https://user-images.githubusercontent.com/88583217/204555672-5f981d17-10b7-4978-b253-8fd27498870c.png)

![image](https://user-images.githubusercontent.com/88583217/204555831-90df37a9-71a7-41b7-89be-8c9a2fe3d334.png)

### Example: Launching Calculator

![image](https://user-images.githubusercontent.com/88583217/204556437-45d998d6-6fd5-48e1-803f-9d17c26106e1.png)

![image](https://user-images.githubusercontent.com/88583217/204556598-13d965b0-3fb6-4ebd-8080-43a7526038b6.png)

The process tree image clearly shows that Internet Explorer is the parent process of calc.

![image](https://user-images.githubusercontent.com/88583217/204556703-7ceb2dbe-b99b-4ad1-adcb-b31c654a5141.png)

## Remote API Function Calls
This procedure has a more complex implementation. To invoke an API remotely, enter a command in the client console using the following format:
```
*Lib_Name!Funct_Name(INT:-123,FLOAT:45.678,LPSTR:"string, with symbols?")
```
- The command starts with the '*' character.
- Lib_Name – library file name (full or relative path) containing the function to call.
- Funct_Name – exported function name in the library.
- The library name and function name are separated by '!'.
- The function name is followed by a parameter list enclosed in parentheses (can be empty).
- Each parameter is represented as a "TYPE:VALUE" pair separated by ':'.
- String constants are enclosed in double quotes.
- Parameters are separated by commas.
- Spaces are not allowed in the command, except within quoted string literals.

Upon receiving a command of this format, the server must validate its syntax before execution. For this validation, a finite-state machine was implemented, whose model is represented as a state transition table.

![image](https://user-images.githubusercontent.com/88583217/204557935-54f660b3-60b3-4a14-9c68-992cc65def06.png)

- q – state of the automaton;
- c – alphabet symbols;
- q=1 – initial state;
- q=0 – terminal state (command is valid);
- q=-1 – invalid transition (command does not match requirements).

The functionality is demonstrated on Windows 10 in the video below.

## Testing on Windows 10

The code injection method itself remains functional without changes on Windows 10 22H2. Only certain demonstration features, such as capturing cmd output after command execution, require minor adjustments. Nevertheless, this did not prevent testing the technique with the current implementation.

https://user-images.githubusercontent.com/88583217/204562303-65425a3d-8dff-4d74-aad2-c7246a7ee757.mp4
