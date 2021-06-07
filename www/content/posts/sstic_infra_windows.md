# Introduction

This post aims to present how to easily setup a lightweight and secure user pwning environment for Windows 10.
From your binary challenge communicating with stdin/stdout, this environment provides a multi-client broker listening on a socket, redirecting it to the IO of your binary, and executing it in a jail.
This environment is mainly based on the project [AppJailLauncher-rs](https://github.com/trailofbits/AppJailLauncher-rs) developed by [Trail Of Bits](https://www.trailofbits.com/). Some security fixes have been added. This post describes also the commands to easily setup the RW rights of the system files from the jail.

The code of our fork of AppJailLauncher-rs is available [here](https://github.com/challengeSSTIC2021/appjaillauncher-rs) and the code of the pwn user windows 10 challenge is available [here](https://github.com/challengeSSTIC2021/Step2_challenge).


## Context 

As a finisher of the previous year challenge, one of Thalium team members participated in the design of the [SSTIC challenge 2021](https://www.sstic.org/2021/challenge_en/).
[SSTIC](https://www.sstic.org/) is one of the most important French cybersecurity conferences. It is held in Rennes every year since 2003, and since 2009 a challenge is proposed several weeks before the start of the conference.
This challenge is usually relatively hard/long to solve: this year, the 12 finishers (out of 1,600 downloads) took from three days (for b2xiao), to more than six weeks.
The challenge was split into five parts, Thalium designed and implemented the second part.
You can find the other parts of the SSTIC challenge on this [GitHub Page](https://github.com/challengeSSTIC2021).

This second step was a user land pwn challenge running on a Windows 10 operating system.
After reversing the binary and finding the vulnerabilities, the players obtained a heap leak and a RW primitive leading to RCE.
For more information about solving the challenge, you can read the solutions of the participants, available on [this page](https://www.sstic.org/2021/challenge/).


## Requirements

The specification can be listed as below: 

* Limited resources, the proposed machine to use for the remote was [16GO DDR3, 2*1 To Storage, Intel® Xeon E3 1220v2 (4 cores)](https://www.scaleway.com/fr/dedibox/start/start-1-l/).
This machine also hosts the remote infrastructure for other steps, and is running with KVM;
* As this challenge is the second step and the first is an easy step, it is possible than dozens of players connect to this challenge simultaneously;
* As it is a CTF challenge, it is mandatory that players can not interfere with other players once they get an RCE;
* Players can not make network connections once an RCE is obtained;
* Be somewhat resistant to *script-kiddies* tentative of DOS;
* Preferable that the binary challenge communicates with stdin/stdout in order to simplify the development of the challenge;
* Private temporary RW folder for each participant.
This requirement is related to the context of the binary challenge;
* Players need to be able to exfiltrate a "big" file on the user Desktop in order to access the next steps of the challenge.



## Considered solutions

The two following solutions were considered:

1. Docker with one of the [windows/servercore/nanoserver base OS images](https://hub.docker.com/_/microsoft-windows-base-os-images) + [socat](https://sourceforge.net/projects/unix-utils/files/socat/1.7.3.2/)/xinetd cygwin to redirect socket to stdin/stdout

2. [AppJaillauncher-rs](https://github.com/trailofbits/AppJailLauncher-rs)

None of these solutions were previously experimented and the lack of remaining time implies that it was not possible to implement then compare the two solutions.


The main drawback of the docker solution is that it seems a little bit expensive in memory, CPU and storage usage at a "high-scale level" compared to the resources available.
Though, nanoserver seems to be very [light](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/system-requirements).


The main drawbacks of the AppJailLauncher solution is that (i) the security of the solution has to be checked, even if it has already been used by other people; (ii) customize some configuration of the environment to have a RW private folder for each participant.

During the conception of the challenge, the point that seems the most important was the following: Does the remote server will still be alive after a "massive arrival" of players on the second step ?
For this reason, the AppJailLauncher solution was selected.
Also, it seemed easier to test this solution as you can work with it on your laptop easily.

Checking if the Docker solution was better than the AppJailLauncher is left as an exercise to the reader :).


# AppJailLauncher-rs

As described on the GitHub page of the project: 

AppJailLauncher is akin to a simple version of xinetd for Windows but with sandboxing enabled for the spawned child processes.
The sandboxing is accomplished via [AppContainers](https://docs.microsoft.com/en-us/windows/win32/secauthz/appcontainer-isolation).
This project is a rewrite of an earlier version in C.

For more information about the AppContainer sandboxing mechanism you can read this [page](https://docs.microsoft.com/en-us/windows/win32/secauthz/appcontainer-isolation).

## Security problems and their mitigations


### Killing processes

After auditing the AppJailLauncher project, it turns out that processes from different players are all spawned with the same AppContainerProfile.
This could lead to a kind of DoS if a malicious player is present.
Indeed, once a player obtains its remote code execution, he could try to continuously kill all the processes containing the name of the binary challenge.
As the duration of the script exploiting the challenge is of several seconds, this malicious player could prevent other people to solve the challenge with a success ratio of 100%.

In order to avoid this kind of DoS, AppJailLauncher has been modified.
In the Thalium's version, each connection creates dynamically an unique AppContainerProfile. The latter is deleted 10 minutes after the connection.

### DoS by huge resource consumption

The AppJailLauncher project does not provide any mechanism to prevent an excessive RAM or processor usage.
Authors of AppJailLauncher propose to add a specific function inside the challenge source code. The latter creates a thread that will kill the current process after a certain amount of time.
Nevertheless, an attacker can bypass this limit easily by killing this thread or spawn a new process.


Windows provides a mechanism to add this kind of limitation, called Job.
In the Thalium's version of AppJailLauncher, a job is set to the spawned process.
This job limits the number of processes that can be launched, the timelife and also the maximum memory usage.
All these information are configurable with parameters given to the command line as described in the section dedicated to the usage of AppJailLauncher.

During the SSTIC competition, the limitations were quite severe: 2 minutes, 100 MB of RAM, and 2 parallel processes.

There is still one problem that is not addressed, a DoS by writing a huge amount of data on the disk.
This was somewhat mitigated by the fact that the only one folder that was writable for each player was removed 10 minutes after its creation.



## Feature needed

Due to the context of the challenge, players need to be able to write to the disk.
This folder needs to be private for each player as the files created are used to exploit the binary.
With the modification of AppJailLauncher, each process is now spawned with a different AppContainerProfile. This is enough to create a folder for each profile and add the RW rights for the SID of the AppContainerProfile.

These private folders were available for 10 minutes before being automatically removed.
In order to be able to reuse this folder during its lifetime, the AppJailLauncher project has been modified and gives to the player a UID when it creates the private folder. This UID is needed in order to retrieve the AppContainerProfile previously created and retrieve the folder name associated with this profile.
For the following connections, the player can give its UID and it will be used to associate the previously created folder to the player.
In order to avoid malicious players bruteforcing this UID, it was hashed together with the IP address of the player, and a secret string to generate the folder name.



## Access to the Desktop file 

After obtaining a remote code execution, players need to exfiltrate a file that is present on the Desktop of a user of the system.
As all the processes are running under different AppContainerProfile, it seems to be a bad idea to add the rights dynamically for each spawned process.

Indeed, it exists an SID corresponding to `APPLICATION PACKAGE AUTHORITY\ALL RESTRICTED APPLICATION PACKAGES` that will be effective for all AppContainerProfile.
This SID is S-1-15-2-2.
The RX rights are needed in order to allow the players to list the folders from C:\ to the file on the Desktop.


## Usage

Command line to add the RW rights to the file on the Desktop of the user, only needed to be executed once per machine:

```
ICACLS "C:\Users" /grant "*S-1-15-2-2:(R,RX)"
ICACLS "C:\Users\Challenge" /grant "*S-1-15-2-2:(R,RX)"
ICACLS "C:\Users\Challenge\Desktop" /grant "*S-1-15-2-2:(R,RX)"
ICACLS "C:\Users\Challenge\Desktop\DRM.zip" /grant "*S-1-15-2-2:(R,RX)"
```



Command line to execute AppJailLauncher: 

```
 C:\Tools\appjaillauncher-rs.exe run --foldermazes "C:\users\challenge\\mazes" "C:\Tools\SSTIC.exe"
```

"C:\Tools\SSTIC.exe" argument is the binary that will be executed when a player connects to the port on which AppjailLauncher listens.

`Foldermazes` parameter is the folder that will contain the private folders.
This folder needs to exist before running AppJailLauncher.

More parameters can be defined, such as the port on which the program listens, the job limitations (time, memory, number of processes), etc.

Remark: During the competition, this command was launched with Powershell.



# Problems to investigate 

Some weird problems happened during the development and the competition.
There are listed below, but not explained, we may investigate it in the future, but if you know why, feel free to contact us :).

## ASLR

On Windows OS, it is known that well-known DLLs (e.g., ntdll) are mapped at the same address for all the processes of the system.
This value is modified at each reboot of the system.
Nevertheless, the mapping of an executed PE, like the binary challenge is known to be different at each execution.
During the development phase of the binary challenge, it appears that executed binaries are always mapped at the same address during a boot life.
This value changes when the system is rebooted.


## cmd Vs Powershell

Most of players obtaining an RCE have launched a process with a call to `WinExec`.
Using this call, some of them have executed `Powershell` and some of them have executed `cmd`.
It turns out that people using `cmd` were not able to list directories and access the file from the Desktop.
Though, people spawning `Powershell` were able to reach the file on the Desktop and read it.
Moreover, people who have spawned a `cmd` terminal then launched a `Powershell` inside, were able to access the file from the Desktop.


## DOS Powershell

It appears that some players found a kind of Denial Of Service by executing some commands with their RCE. Indeed, it seems they where able to make the AppJailLauncher process crash from their jail. It happened two times.

## Powershell arguments

When players executed powershell commands asking arguments from their RCE, the arguments were asked in the `Powershell` terminal from which the AppJailLauncher were launched.


# Conclusion 

Deploying such a CTF infrastructure for a windows pwn user is not well documented on internet.
Two options seem to be interesting (i) Docker and (ii) AppJailLauncher project from TrailOfBits.
Here, AppJailLauncher have been chosen and after some modifications, the obtained solution was satisfying all the requirements.
During the competition, from the 2nd of April to the 20th May, dozens of participants have been connecting to the remote infrastructure.
At least 29 participants exploited the binary and got a remote shell.
None of them have defeated the infrastructure, or maybe we did not see it or maybe nobody tried to defeat it.
The AppJailLauncher has crashed only 3 times, and was restarted with a single command line.
As seen in the *problems* section of this post, there are still weird behaviors that can be related to the AppJailLauncher project.
They need further investigations to determine their origin.


Many thanks to the TrailOfBits team for their AppJailLauncher project.


## Future work

As the time was constrained for the deployment of the infrastructure some things remain to be improved: 

* As it was the first time for the author to read and write Rust, the written code could be slightly improved.
* It seems that some implementations of netcat are not compatible with the UID feature.
The program will still answer that it needs 64 chars exactly even if the player inputs 64 chars.
The problem is certainly due to the mishandling of the end of line character.
* Add some usage statistics (total number of connections, number of crashes, number of live connections, etc.)

