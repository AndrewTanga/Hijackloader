# Hijackloader

- HijackLoader continues to become increasingly popular among adversaries for deploying additional payloads and tooling
- A recent HijackLoader variant employs sophisticated techniques to enhance its complexity and defense evasion
- CrowdStrike detects this new HijackLoader variant using machine learning and behavior-based detection capabilities

CrowdStrike researchers found a type of malware called HijackLoader, which is used by hackers to sneak in other harmful software onto computers. This malware has become more advanced, with new tricks to avoid detection.

One new trick involves hiding itself by pretending to be a regular computer process, and it waits for a specific signal before it activates. This makes it harder for security programs to spot it.

Another trick mixes two different methods to hide the malware, making it even harder to find and stop. The hackers also used additional techniques to cover their tracks and make it difficult for security experts to see what they're doing.

The blog talks about these sneaky techniques used by HijackLoader to avoid being caught by security measures.

# HijackLoader Analysis

## Infection Chain Overview

The malware analyzed by CrowdStrike has a multi-stage process. First, it runs a program called streaming_client.exe, which decodes a hidden set of instructions. These instructions help it find and use certain functions in the system to make it harder to analyze the malware.

Then, the malware checks if the computer is connected to the internet by trying to connect to a specific website using WinHTTP APIs. If it successfully connects, it proceeds to the next step, where it connects to another website to download more instructions. If the first website connection fails, it tries connecting to other websites on a list.
- https[:]//gcdnb[.]pbrd[.]co/images/62DGoPumeB5P.png?o=1
- https[:]//i[.]imgur[.]com/gyMFSuy.png;
- https[:]//bitbucket[.]org/bugga-oma1/sispa/downloads/574327927.png

 After getting the second part of its instructions, the malware looks through the information it received. It specifically looks for the beginning of a PNG image file, marked by certain bytes. Then, it searches for a special code, "C6 A5 79 EA," which tells it how to unlock the rest of the hidden instructions. In this case, the code to unlock is "32 B3 21 A5."
 ![Figure1-2](https://github.com/AndrewTanga/Hijackloader/assets/93886645/45409a94-e19f-4abe-b5d6-c6227b00ebc9)

 
After decrypting and decompressing the configuration, the malware loads a legitimate Windows file named mshtml.dll, which is specified in the decrypted instructions.

Then, it retrieves a special set of instructions called shellcode from the configuration. This shellcode is written into a specific part of the mshtml.dll file and is then executed. This shellcode helps the malware avoid detection and injects more instructions into the cmd.exe program.

The injected instructions perform various activities to avoid detection and inject a final harmful program, in this case, a Cobalt Strike beacon, into another program called logagent.exe. This injection process uses advanced techniques to hide its actions and make it difficult for security programs to detect.

- Process Doppelgänging Primitives: This technique involves manipulating a process's memory to create a duplicate (or "doppelgänger") of a legitimate process, in this case, mshtml.dll. The malware then hollows out (empties) a transacted section within this duplicated process. This section is then utilized to contain the final payload, essentially hiding the malicious code within the legitimate process's memory space.

- Process/DLL Hollowing: In this technique, the malware injects code into a legitimate process or DLL (in this case, mshtml.dll) and then replaces or "hollows out" its original content with malicious code. The fourth-stage shellcode, responsible for executing evasion tactics before passing control to the final payload, is injected into this hollowed-out section within the transacted area established in the previous step. This ensures that the malicious code can execute within the context of the legitimate process without arousing suspicion.

![Figure2a](https://github.com/AndrewTanga/Hijackloader/assets/93886645/136c709e-81ed-43d8-a12a-e865281bcef2)

## Main Evasion Techniques Used by HijackLoader and Shellcode

HijackLoader uses several clever methods to avoid detection. One way is by bypassing hooks, which are security measures, using a technique called Heaven's Gate. It also tries to avoid being caught by security software by "unhooking," which means it remaps certain system files that security programs watch closely.

Additionally, it uses different versions of a method called process hollowing, where it tricks the system into running malicious code by making it look like it's running harmless programs. Another technique it uses is an injection method that combines a few different sneaky tactics, including something called transacted hollowing, which mixes two techniques called transacted section and process doppelgänging with another method called DLL hollowing. These techniques make it harder for security programs to detect and stop the malware.

## Hook Bypass: Heaven’s Gate and Unhooking

Similar to previous versions of HijackLoader, this sample uses a technique called Heaven's Gate to bypass user mode hooks when it's run in the SysWOW64 environment. This technique is similar to how other versions of the malware work, particularly in its use of the x64_Syscall function.

Heaven's Gate is a potent method because it allows the malware to dodge the hooks set up in the SysWOW64 version of ntdll.dll. It achieves this by directly calling the syscall instruction in the x64 version of ntdll, effectively sidestepping the hooks.
Each time the malware calls Heaven's Gate, it provides specific arguments. These arguments are used to pass information or instructions to Heaven's Gate, allowing it to carry out its tasks effectively. The exact details of these arguments may vary depending on the specific functionality being invoked by Heaven's Gate within the context of the malware's operation:
- The syscall number
- The number of parameters of the syscall
- The parameters (according to the syscall)

In this variation of the shellcode, the malware adds another method to bypass hooks that security products might have set up in the x64 version of ntdll.dll. These hooks are typically there to monitor both the 32-bit and 64-bit versions of ntdll.

During this stage, the malware changes the mapping of the .text section of the x64 version of ntdll. It does this by using Heaven's Gate to call NtWriteVirtualMemory and NtProtectVirtualMemory. These calls replace the mapped version of ntdll in memory with the .text section from a fresh copy of ntdll.dll that it reads from the file located at C:\windows\system32\ntdll.dll. This technique helps to remove any hooks that might be present in the system's memory.

This unhooking method is also applied to the process hosting the final payload of Cobalt Strike (logagent.exe), as a final attempt to avoid detection.

## Process Hollowing Variation


To inject the next set of instructions into the child process cmd.exe, the malware employs a common technique known as process hollowing. This involves loading a legitimate Windows DLL, mshtml.dll, into the target process and then replacing its .text section with the malicious shellcode. Another step, crucial for activating the remote shellcode, is explained later.

For setting up this hollowing process, the malware creates two pipes. These pipes are used to redirect the Standard Input and Standard Output of the child process, which is specified in the configuration blob (C:\windows\syswow64\cmd.exe). It does this by placing the handles of these pipes into a structure called STARTUPINFOW, which is then used when spawning the process using the CreateProcessW API.

A notable difference between this implementation and the typical "standard" process hollowing method lies in the way the child process is handled. In standard process hollowing, the child process is usually created in a suspended state, which might raise suspicion. However, in this case, the child process isn't explicitly created in a suspended state, making it appear less suspicious.

Instead, the child process is waiting for input from the previously created pipe. This means its execution is paused, hanging on until it receives data from the pipe. This variation can be termed as an "interactive process hollowing," where the malware interacts with the child process through the pipe, allowing it to control the execution flow without immediately triggering suspicion.

As a result of this setup, the newly spawned cmd.exe process will be reading input from the STDIN pipe, essentially waiting for new commands. At this moment, its EIP (Extended Instruction Pointer) is directed towards the return from the NtReadFile syscall.

The subsequent section outlines the actions carried out by the second-stage shellcode to configure the cmd.exe child process, which will eventually be utilized for the subsequent injections required to execute the final payload.

The parent process, streaming_client.exe, starts by using an NtDelayExecution call to pause, waiting for cmd.exe to complete loading. Once cmd.exe is ready, streaming_client.exe retrieves the legitimate Windows DLL, mshtml.dll, from the file system.

Next, streaming_client.exe loads this DLL into cmd.exe as a shared section. It achieves this using the Heaven's Gate technique:
- Creating a shared section object: This is achieved by making use of the NtCreateSection system call. The malware utilizes this call to create a shared section object.

- Mapping the section in the remote cmd.exe: Following the creation of the shared section object, the malware maps this section into the remote process cmd.exe using the NtMapViewOfSection system call. This enables the malware to share memory between its own process and the cmd.exe process, facilitating the execution of subsequent actions, such as injecting shellcode.

It then replaces the .text section of the mshtml DLL with malicious shellcode by using:
- Heaven's Gate is utilized to invoke NtProtectVirtualMemory on cmd.exe. This call is made to set Read, Write, and Execute (RWX) permissions on the .text section of the previously mapped mshtml.dll section within cmd.exe.

- Following that, Heaven's Gate is again employed, this time to call NtWriteVirtualMemory on the .text section of the DLL. This action overwrites the module, replacing it with the third-stage shellcode.

To activate the injected shellcode, the malware follows these steps:

- Heaven’s Gate to Suspend the Remote Main Thread: The malware suspends the main thread of the cmd.exe process using the NtSuspendThread call via the Heaven’s Gate technique.
- Modification of EIP with a New CONTEXT: Next, it obtains the CONTEXT (register state) of the suspended thread using NtGetContextThread and modifies the instruction pointer (EIP) to point to the location of the previously injected shellcode.
- Heaven’s Gate to Resume the Remote Main Thread: Once the EIP is adjusted, the malware uses the Heaven’s Gate technique again to resume the execution of the main thread in the cmd.exe process with the NtResumeThread call.

However, since cmd.exe is awaiting user input from the STDINPUT pipe, the injected shellcode doesn't execute immediately upon thread resumption. To address this, an additional step is required:

- Write Input to STDINPUT Pipe: The parent process, streaming_client.exe, writes a "\r\n" string (carriage return and newline) to the STDINPUT pipe that was previously created. This action simulates user input, prompting cmd.exe to resume execution and process the input. As a result, the primary thread of cmd.exe resumes execution at the entry point of the injected shellcode.

These steps collectively ensure that the injected shellcode within the cmd.exe process is executed effectively, allowing the malware to proceed with its intended actions.

## Interactive Process Hollowing Variation: Tradecraft Analysis

"Indeed, the activation of the shellcode is reliant on the concept that when a program makes a system call (syscall), the corresponding thread waits for the kernel to respond or return a value. In the context of the described technique:
- CreateProcess: This step involves spawning the cmd.exe process to inject the malicious code by redirecting STDIN and STDOUT to pipes. Notably, this process isn’t suspended, making it appear less suspicious. Waiting to read input from the pipe, the NtReadFile syscall sets its main thread’s state to Waiting and _KWAIT_REASON to Executive, signifying that it’s awaiting the execution of kernel code operations and their return.   
- WriteProcessMemory: This is where the shellcode is written into the cmd.exe child process.
- SetThreadContext: In this phase, the parent sets the conditions to redirect the execution flow of the cmd.exe child process to the previously written shellcode’s address by modifying the EIP/RIP in the remote thread CONTEXT.
- WriteFile: Here, data is written to the STDIN pipe, sending an input to the cmd.exe process. This action resumes the execution of the child process from the NtReadFile operation, thus triggering the execution of the shellcode. Before returning to user space, the kernel is reading and restoring the values saved in the _KTRAP_FRAME structure (containing the EIP/RIP register value) to resume from where the syscall was called. By modifying the CONTEXT in the previous step, the loader hijacks the resuming of the execution toward the shellcode address without the need to suspend and resume the thread, which this technique usually requires.

## Transacted Hollowing² (Transacted Section/Doppelgänger + Hollowing)

- Write the Final Payload: The malware places its final payload in a program called logagent.exe. This program is created by the third-stage shellcode running in cmd.exe. The malware uses a special section to store this payload, making it ready to be used.
- Inject More Code: Next, the malware adds even more code into logagent.exe. It does this by loading another copy of mshtml.dll into logagent.exe and removing its original content. This creates space for the new code.
- Execute More Tricks: The newly injected code performs some fancy tricks to avoid detection. It bypasses certain security measures before finally running the payload stored in the special section created earlier.
  
In simple terms, the malware is carefully preparing logagent.exe to run its final payload without getting caught. It's like setting up a series of hidden traps and distractions to sneak in and carry out its malicious tasks.

## Transacted Section Hollowing

Similarly to process doppelgänging, the goal of a transacted section is to create a stealthy malicious section inside a remote process by overwriting the memory of the legitimate process with a transaction.
In this sample, the third-stage shellcode executed inside cmd.exe places a malicious transacted section used to host the final payload in the target child process logagent.exe. The shellcode uses the following:
- NtCreateTransaction to create a transaction
- RtlSetCurrentTransaction and CreateFileW with a dummy file name to replace the documented  CreateFileTransactedW
- Heaven’s Gate to call NtWriteFile in a loop, writing the final shellcode to the file in 1,024-byte chunks
- Creation of a section backed by that file (Heaven’s Gate call NtCreateSection)
- A rollback of the previously created section by using Heaven’s Gate to call  NtRollbackTransaction

Similar techniques have been observed in other projects that utilize a method called transaction hollowing. This technique involves creating a hollowed-out version of a process and then using a transaction mechanism to insert malicious code into it. This allows the malware to disguise its actions and evade detection. It's like using a disguise to sneak past security measures without being noticed.

Once the transacted section has been created, the shellcode generates a function stub at runtime to hide from static analysis. This stub contains a call to the CreateProcessW API to spawn a suspended child process logagent.exe (c50bffbef786eb689358c63fc0585792d174c5e281499f12035afa1ce2ce19c8) that was previously dropped by cmd.exe  under the %TEMP% folder.

## After the target process has been created, the sample uses Heaven’s Gate to:
- Read its PEB by calling NtReadVirtualMemory to retrieve its base address (0x400000) 
- Unmap the logagent.exe image in the logagent.exe process by using NtUnMapViewofSection 
- Hollow the previously created transacted section inside the remote process by remapping the section at the same base address (0x400000) with NtMapViewofSection

## Process Hollowing

After the third-stage shellcode injects the final Cobalt Strike payload into the logagent.exe process, it proceeds by hollowing out the target process to insert the fourth stage of the shellcode. This fourth stage is crucial for executing the final payload loaded earlier in the transacted section of the logagent.exe process.
Here's how it works:
- The third-stage shellcode, running in cmd.exe, maps a legitimate Windows DLL (mshtml.dll) into the logagent.exe process.
- It then replaces the code in the .text section of mshtml.dll with the fourth-stage shellcode.
- After this modification, the shellcode is executed in logagent.exe using the NtResumeThread function.
- This fourth-stage shellcode performs similar evasion tactics as the third stage did in cmd.exe, ensuring that the final payload is executed without detection.
- In essence, this process involves injecting and executing increasingly sophisticated layers of code to ultimately run the final malicious payload while evading detection.

# CrowdStrike Falcon Coverage

CrowdStrike uses a multi-layered approach to detect malware, combining machine learning with indicators of attack (IOAs). The Falcon® sensor, as depicted in Figure 3, has machine learning features that can quickly spot and stop HijackLoader in its early stages, right after it's downloaded onto a victim's computer. Additionally, behavior-based detection techniques (IOAs) can identify malicious activities as they occur throughout the attack process, including when attackers try to inject processes. This comprehensive approach helps CrowdStrike detect and prevent malware attacks effectively.
![Figure3-1](https://github.com/AndrewTanga/Hijackloader/assets/93886645/f10eb242-f5ed-4dd4-accc-69ff1aa4fa19)

# Indicators of Compromise (IOCs)

    |File                   |SHA256                                                          | 
    ------------------------------------------------------------------------------------------
    |streaming_client.exe   |6f345b9fda1ceb9fe4cf58b33337bb9f820550ba08ae07c782c2e142f7323748|


# MITRE ATT&CK Framework

The following table maps reported HijackLoader tactics, techniques and procedures (TTPs) to the MITRE ATT&CK® framework.
![table 1](https://github.com/AndrewTanga/Hijackloader/assets/93886645/e6c31a00-26b6-4e8c-9955-5717a1a71db3)
![table2](https://github.com/AndrewTanga/Hijackloader/assets/93886645/725656f2-2bdf-46e8-997f-5a735e88ca4f)

