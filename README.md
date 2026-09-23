
# Windows Server 2022 Active Directory Lab

This was a simulation of a small company's network with separate departments. I set up a Windows Server 2022 domain controller and a Windows 11 Enterprise client, then added department-based access, shared folders and enforced password rules via Active Directory and Group Policy.

**Setup:**
- Hypervisor: VirtualBox 7.2
- Domain controller: Windows Server 2022 Evaluation Edition
- Client: Windows 11 Enterprise Evaluation Edition
- Network: Internal Network

**Skills:** 
- Active Directory 
- Group Policy 
- DNS and static IP configuration 
- NTFS and share permissions 
- Access-based Enumeration 
- VirtualBox networking  
- Windows Firewall 
- Troubleshooting

## Installing the VM

![Windows software license error](Windows%20software%20license%20error.png)
To start with it keeps giving me an error saying it does not have the licensing terms. I'm thinking I either do not have the free evaluation version or the file is corrupted. I also read that this could have something to do with the way the vm boots and that removing the floppy disk was a fix for some people. 

I'm going to try these solutions and see how it goes:

- Removing the floppy disk option from the motherboard settings did not seem to work
- Removing the optical drive booting it then adding the optical drive back in and rebooting seemed to do the trick


After booting up the vm I was greeted by the server manager window which I was very familiar with from my labs studying for CompTIA however just in case I ran winver to confirm I was running the right version. Which I was.

![Windows server 2022 confirmation](Windows%20server%202022%20confirmation.png)

There is a countdown in the corner of the screen for when my trial version runs out but I will look into how to extend it later if I need to.



## Setting up active directory

First thing I'm going to do is rename the pc so its easy to identify when looking at other stuff. WIN-Controller-1 will be the new name. When I restart to change the name it asks for a reason which makes sense since servers usually run constantly but I wonder where it logs the shutdown reason stated? 

![New win name confirm](New%20win%20name%20confirm.png)
*New name confirmation*


Looking into this afterwards, I found that Windows logs shutdown reasons in Event Viewer under Windows Logs > System, as Event ID 1074 from the source User32. Filtering the System log for 1074, I found the entry for this restart. It shows the server's old name (WIN-7BMO8DUU9DC), the process that started the restart (the Settings app), the user, and the reason I selected, "Other (Planned)".

![Event ID 1074 for the rename restart](Pasted%20image%2020260923202350.png)

While checking these logs, I also noticed the server's name appears as WIN-CONTROLLER- in some entries. NetBIOS names are limited to 15 characters, and WIN-Controller-1 is 16, so the last character gets cut off. It hasn't caused any problems in the lab, but in a real network I would keep server names to 15 characters or fewer.


After installing active directory I need to name my new forest I think I will call it HLBholdings. I have yet to set up other vms for a multi user simulation however for now I will lay the groundwork and group policy for when I add the other machines. For the OU(organisational unit) I will make one for England and have each team within this OU for the sake of segmentation.

In my company structure I currently have 3 defined teams so I will make OUs for all of these teams.
![OU showcase beginning](OU%20showcase%20beginning.png)
Now I need to add accounts to each of these OUs belonging to the fake employees I have created.
![Accounts in the IT OU](IT%20team%20OU%20accs.png)
*Example of accounts in OU*

All of them currently use a company default password but on next login they will be forced to make a new password. I will also add a security group into each OU that contains all the members of that team. Later I will use this to have specific resources only accessible to specific teams.
![HR security group members](HR%20security%20group%20members.png)
All of the groups are domain local groups just to keep the scope small and not interfere with any other project I startup. 

![Sign-in denied on the domain controller](Sign%20in%20method%20denied.png)
I think the problem here is I'm trying to log in on the Domain controller which is a security risk. Rather than allow the log in which wouldn't be advised from a security standpoint I will setup another machine to act as the client machine.



## Configuring Client machine and joining domain

To test this section I needed a new machine to simulate the clients so I got a windows enterprise evaluation edition
![Windows 11 enterprise evaluation](Windows%2011%20enterprise%20evaluation.png)

Trying to join the computer to the domain came up with this.
![Domain join error](Domain%20join%20error.png)
![Ping test domain](Ping%20test%20domain.png)
Pinging the domain also came up with nothing.

To make things easier I have given the domain controller a static IP address and set the client's to use the domain controller as a dns server.
![Inter vm ping test fail](Inter%20vm%20ping%20test%20fail.png)
To see if the vms could even interact I used the enterprise system to ping the domain controller and came up with nothing which solidifies the issue not necessarily being with the domain itself.

Switching the adapter the vms used resulted in them both losing internet access. Instead of using a bridged adapter I will try to use the host only adapter and see if that fares better. This solution also failed with the ping just being unable to transmit.

![Inter_vm ping success](Inter_vm%20ping%20success.png)
![Inter_vm ping success 2](Inter_vm%20ping%20success%202.png)
After setting the network to internal, making sure both VMs were on the same internal network and changing the client's inbound firewall rules to allow pinging, I got them to ping each other. However, pinging the domain name still failed. Since pinging the IP address works but the name doesn't, this points to a DNS problem rather than a connection problem.

VirtualBox's internal network has no DHCP server, so neither machine is getting an IP address automatically. Joining a domain also depends on DNS, as the client has to look up HLBholdings.local, and only the domain controller's DNS server knows about it. To fix this, I'm going to give both machines static IP addresses on the same subnet (192.254.83.47 for the domain controller and 192.254.199.236 for the client, both with a 255.255.0.0 mask). I'll set the client's DNS server to the domain controller's IP and leave the domain controller pointing to itself (127.0.0.1). After making these changes, the client could ping HLBholdings.local, and I successfully joined it to the domain.

![Domain controller ipconfig output|589](Pasted%20image%2020260923200403.png)
*Domain controller ipconfig*

![Client ipconfig output|591](Pasted%20image%2020260923200411.png)
*Client ipconfig*

Looking back at these settings, 192.254.x.x isn't a private IP range, and the domain controller's default gateway was set to 127.0.0.1, which isn't valid. This worked because the network was fully isolated, but in a real network I would use a private range such as 192.168.x.x and leave the gateway blank when there's no router.

I have successfully logged into David's account.
![David one account](David%20one%20account.png)

I have set a simple password for now however next I will make group policies to mandate password complexity and apply it to all the departments.


## Setting up group policy

To start with I'm going to make a quick list of GPOs and other settings I want to set up:

- Password complexity
- Department wallpapers
- Department shared folders, visible only to each team

### Password complexity

I will start with password complexity as that should be applied to the England OU to cover all departments. 
![Password complexity gpo](Password%20complexity%20GPO.png)
For this I just turned the complexity requirements on and made min length 7. After that I ran gpupdate to make sure the changes take effect
![Pass  complexity](Pass%20%20complexity.png)
Since I don't want anything to override this I also decided to enforce this policy. Now I need to test whether it has worked or not. I will try cake12, which should be rejected if the policy is working. It was rejected with an error saying it did not meet the requirements.

![Pass complexity error](Pass%20complexity%20error.png)

At first I took this as proof that it worked but upon running `Get-ADDefaultDomainPasswordPolicy` I realised that the default domain policy already required a minimum of 7 and complexity. 
![Default domain password policy](Pasted%20image%2020260923185517.png)

To test whether my GPO is actually doing anything, I'm going to change its minimum length to 12 and reset Fatima's password to a 9-character password. If my GPO is working, the password should be rejected. The password was accepted, and the domain policy still showed a minimum of 7. The problem is that the domain controller doesn't use password settings from OU-linked GPOs for domain accounts. It only takes them from GPOs linked at the domain level. 

To fix this, I'm going to link the password complexity GPO to HLBholdings.local, set its link order to 1 so it takes priority over the Default Domain Policy, and remove the old link from the England OU. Re-running `Get-ADDefaultDomainPasswordPolicy` afterwards confirmed the change went through.

![Domain password policy after linking the GPO](Pasted%20image%2020260923194208.png)

For one final test, I'm going to reset Fatima's password to Coffee42x, which meets the default domain policy but is shorter than my new minimum of 12.

![Password reset rejected for being too short](Pasted%20image%2020260923194636.png)

The password was rejected, which proves my change came into effect and my new policy is in use.

### Department wallpapers

The first thing I need to do is share the folder for wallpapers to which I will only give read access for the 3 departments. 
![Wallpaper share permissions](Wallpaper%20share%20delete.png)

While changing permissions for the shared folders I accidently completely denied access to the sysvol folder leading to the client not being able to read the new group policy and subsequently not changing wallpaper. I realised this after running gpupdate on the client computer and it failing due to not having read access to the sysvol folder.

To fix it, I went back to the SYSVOL permissions and removed the Deny entry I had added, which restored read access. After that, gpupdate on the client ran successfully. I didn't record exactly which permission I had changed at the time, but the lesson was clear: in Windows, an explicit Deny overrides any Allow, so a single Deny entry can block access even when other permissions grant it. Since then, I've been more careful to check which folder I'm editing, and to use Deny sparingly.

Then I need to create a wallpaper hr GPO and copy the path to the wallpaper I want the hr team to have. After a gpupdate the hr teams accounts should have the correct wallpaper which it does.
![HR wallpaper applied|439](HR%20wallpaper.png)
Next I will do this for the rest of the teams.

The rest were done smoothly with a small blip due to the last file being png and I entered it as a jpg file. 

### Department shared folders

Last thing to do for this section is to setup shared folders for departments. To start I'm going to make a root folder called Company folders and share it. Going to server manager > shares I can enable Access-based enumeration which will make it so that anyone without read permission for the folders in the share will not be able to see it. I will disable inheritance on the company folders then make a resources folder for each department and disable inheritance on them aswell. After removing authenticated users from the permissions panel and using the security groups I made to apply permissions for each folder.
![Hr resources permissions](Hr%20resources%20permissions.png)
Here is an example where I gave the hr team permissions to modify the folder but not delete the base folder to avoid complications.

![resources folders](resources%20folders.png)
Despite all of these folders being under the company resources share if we look from Fatima's account we will only see the HR folder as she is part of the hr security group and no others.

![HR share](HR%20share.png)

Now I will go and do the same for the rest after which this section will be done. While doing this I encountered a problem that users could not make or delete folders within their resources folder. After using effective access via NTFS to see what was limiting their ability I found that the share permissions needed to be altered, so I gave change permission on a share level to authenticated users and assigned permissions per folder. Doing this Fatima can create folders and files as well as delete them but not delete the HR resources folder itself.

Having gone through given all the permissions and checked each user has the correct controls this section is done.



## Problems and fixes

| Problem                                               | Cause                                                                                | Fix                                                                                                    |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| Installer said it couldn't find the licence terms     | VM boot issue with the optical drive                                                 | Removed the optical drive, booted, re-added it and rebooted                                            |
| Couldn't sign in as a user on the domain controller   | Standard users aren't allowed to log on to a domain controller                       | Set up a separate Windows 11 client                                                                    |
| VMs couldn't ping each other                          | Wrong adapter type, and the client's firewall blocking ping                          | Switched to an internal network and allowed inbound ping on the client                                 |
| Client couldn't find or join the domain               | Internal network has no DHCP, and the client couldn't look up the domain through DNS | Gave both machines static IPs on the same subnet and pointed the client's DNS at the domain controller |
| Password policy GPO had no effect on domain accounts  | Password settings in OU-linked GPOs aren't used for domain accounts                  | Linked the GPO at the domain level with link order 1                                                   |
| Group Policy stopped applying on the client           | I accidentally added a Deny permission to SYSVOL                                     | Removed the Deny entry                                                                                 |
| One team's wallpaper didn't apply                     | File was a PNG but the path said JPG                                                 | Corrected the file extension                                                                           |
| Users couldn't create or delete files in their folder | Share permissions were more restrictive than NTFS                                    | Gave Change at share level and kept access control in NTFS                                             |
