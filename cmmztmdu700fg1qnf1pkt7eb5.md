---
title: "Hypervisor cracks and Windows security - an introduction by RessourectoR on cs.rin.ru"
datePublished: 2026-03-21T04:21:30.897Z
cuid: cmmztmdu700fg1qnf1pkt7eb5
slug: hypervisor-cracks-and-windows-security

---

<div data-node-type="callout">
<div data-node-type="callout-emoji">💣</div>
<div data-node-type="callout-text">I believe everyone should be aware of these new era of bypasses for Denuvo and their pros and cons, so I am replicating the forum posts here. You can find the original thread <a target="_self" rel="noopener noreferrer nofollow" class="text-primary underline underline-offset-2 hover:text-primary/80 cursor-pointer" href="https://cs.rin.ru/forum/viewtopic.php?t=156407" style="pointer-events: none;">here</a> and <a target="_blank" rel="noopener noreferrer nofollow" class="text-primary underline underline-offset-2 hover:text-primary/80 cursor-pointer" href="https://cs.rin.ru/forum/viewtopic.php?t=156419" style="pointer-events: none;">here</a>.</div>
</div>

### **Related topics**

*   **EDUCATION:** (This topic) *\- Basics to help you understand security implications so that you can decide if you want to use hypervisor cracks.*  
    **INFO:** [Hypervisor cracks - best practices and release requirements](https://cs.rin.ru/forum/viewtopic.php?t=156419) *\- How to use hypervisor cracks responsibly and quality requirements for approved crack releases.*  
    **SUPPORT:** [Hypervisor cracks support](https://cs.rin.ru/forum/viewtopic.php?t=156435) *\- Support for all approved hypervisor crack releases, report issues and ask for help here. Please use this topic and do not ask in game topics. If this topic is locked, patiently wait for updates and do not spam the forum.*  
    **DISCUSSION:** [Hypervisor cracks discussion](https://cs.rin.ru/forum/viewtopic.php?t=153951) *\- For feedback on this post, the other posts linked above and all other talk. Do not ask for support here.*
    

*Disclaimer: This is researched and phrased to the best of my abilities. I am reasonably qualified to write a post like this, but I am not an expert in virtualization and I have very little reverse engineering knowledge. I also don't use Windows regularly. If you find inaccuracies or incorrect information, please let me know and I will correct it.*

This is a high-level, educational guide on

*   what a hypervisor is,
    
*   how Windows uses one to enhance system security,
    
*   how a novel class of Denuvo cracks uses one to emulate a system that passes the most difficult Denuvo checks,
    
*   why these two use cases cannot (easily) coexist,
    
*   and the practical impact of disabling all that Windows security.
    

### **What is a hypervisor?**

In the context that is relevant to us here, a [hypervisor](https://en.wikipedia.org/wiki/Hypervisor) is a software that allows multiple operating systems to run on the same computer, by dividing hardware resources into so-called [virtual machines](https://en.wikipedia.org/wiki/Virtual_machine) (VMs). Software like VirtualBox, VMWare or Hyper-V can be installed on your system like normal programs and you can use them to create virtual machines in which you install different operating systems as "guests", strongly isolated from the "host" OS and each other.  
It's intuitively clear that putting an OS into a virtual machine, with it's own virtual CPU cores, reserved memory and no direct access to the hardware firmware or your personal files can be a very strong protection against malicious software running inside the VM.

A bare-metal hypervisor can go one step further: It does not run as an application in your OS, but runs directly on the hardware, so that even your main OS accesses hardware resources through the hypervisor. It is [assisted by virtualization features of your hardware](https://en.wikipedia.org/wiki/Virtualization#Hardware_virtualization), such as SVM/AMD-V and Intel VT-x, to make this more efficient and allow your OS to load a hypervisor on boot that takes over hardware resource management from the OS. This hypervisor is then able to run other OSes, specialized security software or even another hypervisor completely isolated from even the main OS that you booted into.

### **Windows virtualization-based security components**

On modern systems with Secure Boot, TPM 2.0 and hardware-assisted virtualization capabilities, Windows 10 and 11 enable, mostly\* by default, various security solutions via [Virtualization-based Security (VBS)](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/oem-vbs). VBS is an umbrella term for using a bare-metal hypervisor, the Windows hypervisor, to create isolated virtual spaces that are safe from even a fully compromised OS, in which these security components run and monitor the OS or store confidential information.

The following Windows components are such security solutions:

*   [Memory Integrity (HVCI)](https://learn.microsoft.com/en-us/windows/security/hardware-security/enable-virtualization-based-protection-of-code-integrity): Runs checks to detect malicious or at least unexpected modifications of Windows kernel code and restricts suspicious kernel memory allocations. For example, I imagine this could protect against malicious software that is being run with administrative privileges and attempts to modify system files, or against memory security vulnerabilities in user-run applications.
    
*   [Credential Guard](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/oem-credential-guard): Stores access credentials, such as passwords, authentication data, biometric data etc. in an isolated environment.
    
*   [Windows Hello](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/windows-hello): Allows you to log in with convenient methods like a short PIN, facial recognition or fingerprint scan. I have not found a direct source for this, but it probably prefers Credential Guard to store its highly sensitive data. The login methods it provides tend to break when some of the components above are disabled. It is also protected by System Guard, if that is enabled.
    
*   [System Guard (Secure Launch)](https://learn.microsoft.com/en-us/windows/security/hardware-security/how-hardware-based-root-of-trust-helps-protect-windows): An advanced system hardening framework that protects the OS boot process and [System Management Mode](https://en.wikipedia.org/wiki/System_Management_Mode) (SMM, commonly used by the BIOS to run hardware configuration software) from (arguably sophisticated) rootkits. Such rootkits could compromise the hypervisor itself, so this protection is assisted by various hardware security features of modern processors. Backed by TPM 2.0, this also allows to monitor system integrity, including the other security components mentioned here, after boot continuously and verify it from a remote system.  
    From what I could find, this is cutting edge and not enabled by default.
    

\* Even though hardware and boot requirements are met, Windows sometimes seems to fail at enabling features that are supposed to be enabled automatically, such as VBS and memority integrity.

\*\*Without the Windows hypervisor, none of these security features can be used. By design, the hypervisor cannot be disabled directly. Instead, all the above features that want to utilize VBS signal that it needs to be enabled, which then loads the hypervisor. Therefore, we must disable all those features to prevent the Windows hypervisor from being loaded.

A boot option that prevents Hyper-V from loading the hypervisor also needs to be added.\*\*

### **Modifying system behavior with a bare-metal hypervisor**

A bare-metal hypervisor controls all access of the OS to the CPU, memory and all other hardware, which means the hardware environment could be spoofed and OS system operations could be manipulated by it. This is a great power to have against a copyright protection that can defend itself against many known techniques from other software running within the same OS, such as debuggers, emulators or memory patchers. Essentially, the hypervisor crack method includes a Windows kernel driver that inserts itself as a very simple hypervisor, whose main job is to fool Denuvo checks.

Recent Windows versions refuse to load kernel drivers that are not approved and cryptographically signed by Microsoft through WHQL. This is called driver signature enforcement (DSE). Only legitimate companies with representatives that undergo extensive identity checks for a [special certificate](https://learn.microsoft.com/en-us/windows-hardware/drivers/dashboard/code-signing-reqs#where-to-get-ev-code-signing-certificates) have a chance at getting their drivers signed, **which means that we must disable DSE to load our custom driver.**

**Why not both?**

If the Windows hypervisor sits between the OS and the hardware and supports all the hardware capabilities that Windows needs, why can't we just put our anti-Denuvo hypervisor between the Windows hypervisor and Windows? Hypervisorception!  
Technically, this is possible and is called [nested virtualization](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/nested-virtualization), but Microsoft decided that this is only supported for other Windows hypervisors in Hyper-V VMs and not just any third-party hypervisor. The Windows hypervisor simply does not pass through the hardware-assisted virtualization features to the OS. Even if it did, it is questionable whether it would just allow another bare-metal hypervisor between itself and the OS - nested virtualization is only discussed in the context of virtual machines, not stacking multiple bare-metal hypervisors on top of each other.

Other virtualization software like VMWare [is being forced](https://techdocs.broadcom.com/us/en/vmware-cis/desktop-hypervisors/workstation-pro/25H2/using-vmware-workstation-pro/running-workstation-on-a-hyper-v-enabled-host.html) to [use the Windows hypervisor](https://learn.microsoft.com/en-us/virtualization/api/#windows-hypervisor-platform) instead of their own, or have their users disable all the above listed features so that the Windows hypervisor is no longer loaded and VMWare can directly access the hardware-assisted virtualization features.  
Another example is Virtualbox, which seemingly tries to work without hardware-assisted virtualization and [is extremely slow](https://www.virtualbox.org/manual/topics/Troubleshooting.html#ts-hyperv) as a result.  
So even "legitimate" software for hosting virtual machines that does not aim to run bare-metal has major issues.

**This means that we have no choice but to get rid of the Windows hypervisor and with it, all those security components.**

### **I want to play that new Denuvo-protected game, is it safe to disable all this and use a hypervisor crack?**

*There is no simple answer. This is just my personal take as someone with 10 years of experience in security-focused system administration and only a casual interest in gaming.*

It's true that the most common threats are info stealer malware from fake download buttons, ransomware that encrypts your files or joining a DDoS botnet. I suppose such malware is usually not interested in higher privilege escalation or hardware sabotage, if it can already access what it needs. It's also true that the best protection against such malware is a good ad blocker, staying on trusted sites and user education.  
More experienced PC users develop a false sense of security from seeing how successfully they avoid malware infection by "being smart". They argue that they don't need all these restrictive, patronizing security features and AVs that just annoy with false positives, because their malware-free track record "proves" that they know better. They also argue that more advanced threats are not aimed at home users, but corporate networks and too unlikely to care about. Especially relevant for gamers: Virtualization can reduce system performance and whether that is noticeable or not is also a point of contention.

The other side of the argument: You would be disabling technology that evolved from decades of security research, ignoring what experts consider necessary nowadays. You would willingly give up protections against common classes of software vulnerabilities and if you ever do get a more advanced malware, it can breeze right through so that your PC can stay part of a botnet for eternity or spread to more local network devices more easily. If a lot of people remove these protections - especially DSE and memory integrity - for gaming, one of the main use cases of Windows PCs at home, it might be worth the effort for malware authors to target such setups. Widespread usage of HV cracks could encourage manipulated fake releases, because people who download those can be expected to disable all protection, including AV exclusion. The knowledge and effort required to take precautions and verify files properly is higher than with common threats and the potential consequences much more severe.  
Aside from the disabled Windows features, even if you trust the authors of the hypervisor driver and even compile it yourself from source, a serious vulnerability in its code could instantly provide maximum and undetectable access to your system.

Whether that game is worth the risks is something you will ultimately have to decide for yourself.

*However, if you do not understand the general concepts described in this guide or scrolled down here without reading it, you are most certainly not equipped to make this decision.*

### A note on Meltdown software mitigation and hypervisor drivers

As of March 2026, the hypervisor drivers released by the team from the Mkdev discord (DenuvOwO, hvteam) conflict with KVA Shadow, Microsoft's software mitigation for Meltdown, the speculative execution side channels vulnerabilty found mostly in older Intel CPUs (before 2019). KVA Shadow is incompatible with hvteam's syscall hook implementation and needs to be disabled, I quote: hvteam wrote:

> Our LSTAR hook will not be mapped into the memory space in user mode page tables with the mitigation active, which will cause a page fault upon execution of a SYSCALL instruction resulting in a UNEXPECTED\_KERNEL\_MODE\_TRAP bug check (BSOD). To prevent this, our driver will not load if this mitigation is active and will return STATUS\_HV\_FEATURE\_UNAVAILABLE.

The script included with the releases handles this.