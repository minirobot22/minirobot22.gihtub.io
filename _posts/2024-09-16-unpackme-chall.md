
---
title: [Uncovering New Techniques through solving "UnpackMe" Challenge]
date: 2024-03-16 00:00:00 +0800
categories: [Digital Forensics]
tags: [dfir,ir,malware analysis]     # TAG names should always be lowercase
img_path: /assets/images/unpackme/
---



As a malware analyst, we received a sample 
Often times as a malware analyst, I receive vague file samples and get asked to analyze them. What I mean by vague is that they either unknown to most Public threat intelligence platforms or they do exist with low/high detection rate but with generic detection names. Those are mostly hueristix detections which.. 
In all cases, I usually perform the analysis with the following goals in mind:
. Determine if the sample is actually doing something harmful, that is true or false positive.
. The second most important thing to be concerned about is the malware capabilities & the risk associated with it

In our current scenario, we are mostly going to uncover & highlight some of the main capabilities of the sample and answering the challenge questions as we progress.
Along the way, we will learn various techniques and work arounds to tackle some of the challenges encountered that hindered our dynamic analysis of the malware.
First of all, I`d like to examine my sample using basic static analysis, throwing the sample on Pestudio, DIE for quick triage.
Will not spend too much time in this phase as the sample seems to be packed, no visible strings, entropy seems to be high which indicates some degree of packing.
[Entropy]([1].Entropy.png)
_Entropy_
We saw VirtualAlloc as one of the API calls being imported from the imports address table of the sample, so,
Load the sample in x32dbg, and let`s simply start by adding a breakpoint on VirtualAlloc
[2]
After we run the program, we hit our breakpoint
[3]
Continue until we return back from the system dlls (kernal32,etc), and observe the address returned in EAX
[4]
Allocated space by VirtualAlloc is currently empty, stepping over a few instructions, particularly after the next call (call 763E52 in my case), the memory address started to populate, this memory segment will contains the unpacked binary that the malware execution will be transferred to.
[5]
Scrolling down a bit, we can see the start of an executable file (MZ header).
[6]

we can follow on memory map, then right-click on the specified memory region and dump it to a file
[7]
Open the dumped file in any hex editor and get rid of the extra bytes before the MZ header, then save the file
[8]
Now, we should have the clean unpacked sample, and to make sure, we can inspect its imports and strings again. In fact, from the strings alone, we have a pretty good idea about what the malware sample is doing( information stealer) as we see below
[9-1] [9-3]
Among other things, strings also reveal the internal path of the project build 
[9-2]
Furthermore, if we search for some of the unique strings it would give away the sample varient which is one varient of RacoonStealer
Now, let`s start examining the malware by Loading he unpacked sample into IDA. The malware begins by creating a mutex (function sub_DD2DF7) to prevent the malware from being executed twice on the same host.
[10]
We can see the API OpenMutexA inside the function, with esi register holding the mutex name.
[11]
If we debug the sample using IDA, and we examine the address at ESI, we can obtain the Mutex name as below
[12]
[13]
Next, the malware calls (435AE5) to check if the process is running with LOCAL SYSTEM privileges, if so, it will proceed by finding explorer.exe process, duplicate its token and execute with its privileges.
[14]
Here is the portion within the function (435B8A) where it is trying to find explorer.exe 
[15]
Next up, the execution will continue to find the local machine language by calling GetUserDefaultLCID and GetLocaleInfoA, 
[16]
Then it dynamically uses XOR in a bunch of loops to check a set of languages (Russia, Kazakhstan, Uzbekistan, etc) if they match, the malware will stop execution and exit.
The malware uses repetitive XOR blocks to decrypts each segment during execution. At the block location (loc_429AE3), before jumping into a different block of code that appears to have some base64 strings, it reveals a special string after finishing the XOR loop.
[17-1] [17-2]
This will turn out to be the RC4 key it uses for encrypting all C2 communications.
In the next block, we see three base64 strings
[17-3]
Stepping over until we hit function (DB47F6), after the function call, we immediately notice a C2 domain pops up
Ttttttt.me
[17-3]
Let`s step back for a moment and look back before the function gets called, the function had two parameters that were pushed to the stack, one is the base64 decoded value of the first base64 string shown previously, the other is the RC4 Key that we discovered.
[17-4]
Therefore, we know that this function is using the decoded string with the RC4 key to decrypt its first C2.
If we look back even further, we can get the function that performs the base64 decoding, let`s rename & label those two important functions.
 
[18.0 and 18.1]
Digging deeper into the RC4 Decryption function, we notice 4 subroutines, from there we can identify the RC4 algorithm by checking both functions (41468E) and (414712)
[19] and [20]
Above screenshots clearly shows the key Scheduling Algorithm (KSA) generation  of the RC4 (the 256 loop is a quick giveaway), then the key stream is being XORed with each byte of the payload in the other function.
Going further after RC4 decryption of the domain, we should expect to see some network communication to that C2 domain, this takes place at function (430F27) as we see below 
[20.1] and [21]

Next function appears to be checking for certain strings in the response body of the HTTP request. We clearly see some html tags "description and dir=auto>
[22.0]
[22.1]
In order to make sense of it, we checked the C2 link that the malware was trying to request () , then looking for the html strings needed, nothing was found. It appeared that the C2 was not active anymore.
[Telegram photo on my personal laptop]
As a result of that, the malware was stuck at the next block of code, Which is a loop that sleeps for 5 sec. , then continues to request the domain until the string match is found.
[22.2]
Apparently, this hindered our dynamic analysis as the malware was not able to retrieve this string to continue its operation.
So, we have to come up with a different approach to emulate this on our local network to further continue debugging and discover the rest of the stealer operation.


MALWARE C2 EMULATION
---------------------------------------
So far, we`ve understood how the malware uses RC4 encryption to decrypt its C2 communications, and we have also seen that the base64 string () was actually the telegram C2 domain after decoding and decrypting it. In order to continue debugging, we have to alter the code a bit and replace this base64 string with our own that would resemble our own C2 on a local network.
To do that, let`s setup an http server on our localhost, prepare similar C2 page by copying the telegram html code to our page, and add the missing html strings that the malware would expect to retrieve as below.
[23]
Mine would be saved and hosted at localhost/telegram.html
knowing the RC4 key, we can assemble our base64 with the help of CyberChef as below
[23-0000-CyberChef]

Now, let's open the malware file with hex editor and search for the base64 string (qSVdAbi/K2pP9eTPjNld5MgaAL+bQsyox4MDv0iVTuA=), our goal is to replace that portion with our own base64 so that after malware decodes it, we get our telegram.html.
[23.00-Photo with HexEditor]
Write-click, then choose paste write and save the file.
Let`s test it by going to the instruction () and continue debugging, we should see our telegram page being resolved. 
[pic]
Now, if we continue debugging from where we left off, we should jump at the block after the HTTP request, and our embedded string should appear as shown below
[23.1]
Next, if we step over a few instruction, we would start to see the string being filtered.
[23.2]
If we look closer after the function that extracts the string, we would see two calls to the same function (renamed filter_out here). In the first call, the value 5 is being pushed into the stack before the function call, and value 6 in the second function call, which corresponds to filtering out the first 5 characters of the strings, and the last 6 characters respectively.
[23.3]
Here is the code inside the function
[23.4]

Now, let