\# SOC Home Lab – Attack Simulation \& Detection with Sysmon + Splunk



This is my first SOC home lab. The goal was pretty simple: set up an isolated lab environment, attack my own Windows machine from Kali, and then see what that attack actually looks like in the logs. I wanted to understand what telemetry gets generated when something bad happens on an endpoint, and how to actually go find it in Splunk.



\## What I built



Two VMs in VirtualBox:

\- \*\*Windows 10\*\* (target) – `192.168.20.10`

\- \*\*Kali Linux\*\* (attacker) – `192.168.20.11`



Both VMs are on their own isolated internal network in VirtualBox (I named it `homenetwork`), with no internet access and no access to my host machine or LAN. I statically assigned IPs to both so they could talk to each other without relying on DHCP. I did this specifically because I didn't want any malware I ran to have a way out of the lab.



On the Windows side, I installed and configured Sysmon for detailed endpoint logging, then installed Splunk and set it up to ingest Sysmon logs along with the Application, System, and Security event logs.



\## Setting up the network



After putting both VMs on the internal network, I gave them static IPs and confirmed everything with `ipconfig` on Windows and `ifconfig` on Kali.



\*\*Windows – 192.168.20.10\*\*

!\[Windows static IP](screenshots/static\_ip\_config\_check\_windows.png)



\*\*Kali – 192.168.20.11\*\*

!\[Kali static IP](screenshots/static\_ip\_config\_check\_kali.png)



Then I tested connectivity between the two. Pinging Kali from Windows worked fine. Trying it the other way around (Kali pinging Windows) didn't work at first because Windows Firewall blocks inbound ICMP by default, so I just confirmed the link using the Windows → Kali ping instead.



!\[Ping check between machines](screenshots/ping\_check\_btw\_machines.png)



\## The attack



\### Recon

First thing I did from Kali was run an Nmap scan against the Windows machine to see what was open:

nmap -A 192.168.20.10 -Pn

All 1000 scanned ports came back filtered, meaning the target wasn't responding to any of the probes — Windows Firewall was still fully up at this point (I hadn't disabled Defender/Firewall yet). Interesting result on its own, since it shows the scan itself is visible even when nothing is actually open.



!\[Nmap scan](screenshots/nmap\_scan.png)



\### Building the payload



Next I built a simple reverse shell payload using `msfvenom`, using the Meterpreter reverse TCP payload for Windows x64, pointed back at my Kali IP on port 4444, and saved it as `resume.pdf.exe` (I wanted it to look like a normal file if someone wasn't paying attention to extensions).



!\[Creating the malware with msfvenom](screenshots/creating\_malware.png)



Just to confirm what I actually built, I ran `file` on it to check the binary type — confirmed as a PE32+ Windows executable.



!\[Checking file type](screenshots/file\_type\_of\_malware.png)



\### Setting up the listener



On the Kali side, I opened up Metasploit and set up a multi/handler to catch the reverse connection once the payload got executed.



!\[Using exploit/multi/handler](screenshots/use\_exploit.png)



I configured the payload to match what I generated with msfvenom (windows/x64/meterpreter/reverse\_tcp) and made sure LHOST/LPORT matched what was baked into the payload.



!\[Configuring Metasploit payload options](screenshots/configuring\_metasploit\_1.png)



Then set LHOST to my Kali IP and started the handler, so it's now sitting there listening for a connection.



!\[Starting the exploit handler](screenshots/starting\_exploit.png)



\### Delivering the payload



To get the file over to the Windows machine, I just started up a quick Python HTTP server on Kali in the same folder as the payload.



!\[Hosting the malware with Python's HTTP server](screenshots/host\_malware\_python\_server.png)



Then from the Windows machine, I browsed to Kali's IP on port 9999 and downloaded `resume.pdf.exe` from the directory listing.



!\[Downloading the malware from the browser](screenshots/access\_c2\_server\_download\_malware.png)



\### Execution



After running it on Windows, I checked Task Manager and could see `resume.pdf.exe` actively running under PID 5484.



!\[Malware running in Task Manager](screenshots/malware\_running\_taskmanger.png)



I also ran `netstat -anob` from an admin command prompt to confirm there was an actual established connection back to Kali on port 4444, tied to that same PID.



!\[Netstat showing established connection](screenshots/netstat\_response\_connection\_established.png)



Back on Kali, the handler caught the connection and I had a live Meterpreter session. Dropped into a shell from there.



!\[Gaining the reverse shell](screenshots/gaining\_reverse\_shell.png)



From the shell, I ran a few basic post-exploitation commands just to generate more telemetry — `net user`, `net localgroup`, and `ipconfig`.



!\[Running commands through the shell](screenshots/use\_shell\_to\_run\_cmds.png)



\## Finding it in Splunk



This is the part I was actually most interested in — going back and finding all of this in the logs.



Once Sysmon events were flowing into my `endpoint` index, I could search for the process creation event and see that `resume.pdf.exe` was the parent process that spawned `cmd.exe`.



!\[Splunk showing resume.pdf.exe spawning cmd.exe](screenshots/splunk\_1.png)



From that event I pulled the process GUID for the cmd.exe process (PID 8308), which I used to pivot and pull every event tied to that exact process instance instead of relying on the PID (which can get reused and give you false matches).



!\[Splunk process GUID and hash details](screenshots/splunk\_2.png)



Using that GUID, I built a table view of the full chain: `resume.pdf.exe` → `cmd.exe` → `net user`, `net localgroup`, `ipconfig`. This basically lays out the entire attack from execution to post-exploitation commands, in the right order, using just the Sysmon telemetry.



!\[Full attack chain traced via process GUID](screenshots/splunk\_use\_guid\_complete\_attack\_.png)



\## What I learned



\- Process GUID is way more reliable than PID for tracing what a process actually did, since PIDs get reused constantly on Windows.

\- Even a "no results" Nmap scan (all ports filtered) still shows up clearly in telemetry — the absence of open ports doesn't mean the absence of evidence.

\- Disguising a file as a PDF (via the filename) does nothing at the binary level — `file` and Task Manager both show exactly what it is once it's running.

\- Splunk not knowing what to do with Sysmon logs out of the box was a good reminder that ingestion isn't the same as usable data — you need the right index and field parsing (Splunk Add-on for Sysmon) before any of this is actually searchable.





