---
title: "PenTesting PowerShell Remoting"
datePublished: Sat Feb 27 2016 06:30:00 GMT+0000 (Coordinated Universal Time)
cuid: clzwnoskt000009mmdo3k11w9
slug: pentesting-powershell-remoting
tags: security, pentesting

---

I have been planning to kick off my blog for sometime now and has just not happened, until now. I intend to share my experiences and notes mostly i take while I learn something. My first blog is about an article I wrote for [PenTest Magazine](https://pentestmag.com/magazines/) for their PowerShell edition, published in January 2016.

PowerShell has been around for quite some time now. Taking humble steps, from a downloadable edition of PowerShell v1.0 for Windows XP, SP2 & Windows 2003 to becoming available by default starting Windows 2008, indeed it has come a long way yet going stronger! I am very sure we must have used this blue screen command line interface sometime in our lives, it could even be just hitting the ‘ls’ command 🙂 A Linux inclined friend of mine uses PowerShell for this, funny right 🙂

Today, let’s talk about one of coolest features of PowerShell -Windows Remote Management (WinRM), which became available from Version 2.0. I am assuming most of us are already familiar around this topic, establishing remote connections and exercising commands such as Enter-PsSession, Get-PsSession etc. In case, you are not, I am anyway covering the basics around it and you can follow on

WinRM 101:

[Windows Remote Management](https://msdn.microsoft.com/en-us/library/aa384426(v=vs.85).aspx) (WinRM) is a ‘service’ providing Remoting features over the WS-MAN Protocol ([WS-Management Protocol](https://msdn.microsoft.com/en-us/library/aa384470(v=vs.85).aspx)). This framework was designed to be a secure and reliable method for managing computer that’s built on Simple Object Access Protocol (SOAP) and HTTP

Some of you may ask, why Windows Remoting while we have WMI and VB scripting that could get us similar functionality. We often see that something that works perfectly fine in our local system fails badly when executed in remote systems, this is because that particular API can only be used locally and is not available on the remote machine. PowerShell on the other hand does not rely over the programing interfaces to talk to remote computers. It simply connects the local Windows PowerShell Session to another Session running on the remote system. The commands that are executed are sent to the remote computer and executed locally there and then the results are sent back. We can establish similar sessions across multiple computers at the same time. We can connect to as many computers as we want, while Microsoft mentioned they tested with 25000 computers. However, commands executed are run in 32 computers by default considering the overhead it may have on the system, this is still configurable if needed. Windows Remoting is turned on by default for Windows 2012, Servers prior to this needs to be enabled manually and so do the client machines.

Considering Remoting to be enabled in all client machines, we can push a Group Policy to ease this task. Refer the this [article](https://www.penflip.com/powershellorg/secrets-of-powershell-remoting/blob/master/configuring-remoting-via-gpo.txt) for detailed instructions to do this. Having said that, let me walk you through the steps to configure a 1-1 communication between Windows Server 2012 and a client running Windows 10 and explore the weakness and hardening mechanisms.

Quick & Default configuration**:**

**Enable-PSRemoting**

![Enable-PSRemoting](https://threatpointer.blog/wp-content/uploads/2016/02/enable-psremoting.png?w=828&h=523 align="left")

Enable-PsRemoting: This command is going to perform a quick default configuration of Windows Remoting. As shown in the above screenshot it explains the things it does. We can run the same command on both the ends and should enable Remoting. We can establish a connection using the command ‘Enter-PsSession’ and takes an argument -Computername (this will be the remote computer) in my case it will be pos

![pos](https://threatpointer.blog/wp-content/uploads/2016/02/pos.png?w=390&h=49 align="left")

If everything goes well, you will get to see something as shown in the above figure. This is a remote shell that could be used for any given administrative task. That being said, the questions that we need to ask ourselves is what happened in the background that gave us a shell, we never used a username or a password.. Let’s take a step back and look at the default configuration & imagine what could go wrong and how to better it.

We have something called as [PowerShell Providers](https://technet.microsoft.com/en-us/library/ee126186%28v=vs.85%29.aspx?f=255&MSPPError=-2147217396) and simply put they act as a drive in this context. Upon, executing this command we should see a drive for wsman and we could cd into it. We can explore the directories here and should give us a complete picture of the settings that are enabled by default.

![Get-PsProvider](https://threatpointer.blog/wp-content/uploads/2016/02/get-psprovider.png?w=849&h=395 align="left")

Now that we understand how Remoting works and with the awareness of the configuration settings, let me point you to some of the areas where we should lookout while assessing Windows Remoting.

**Authentication:** Windows Remoting supports 6 types of authentication. When we were testing our connection during the configuration steps, we used Kerberos authentication. This is default when we do not specify the authentication scheme and attempted to connect computers.

![Authentication](https://threatpointer.blog/wp-content/uploads/2016/02/authentication.png?w=847&h=109 align="left")

**The Double Hop problem**: An administrator uses PowerShell Remoting to connect to Server A and then attempts to connect from Server A to Server B. Unfortunately, the second connection fails. As a workaround, PowerShell provides the CredSSP (Credential Security Support Provider) option. When using CredSSP, PowerShell will perform a “Network Clear-text Logon” instead of a “Network Logon”. Network Clear-text Logon works by sending the user’s clear-text password to the remote server. When using CredSSP, Server A will be sent the user’s clear-text password, and will theref[ore](http://www.powershellmagazine.com/2014/03/06/accidental-sabotage-beware-of-credssp/) be able to authenticate to Server B. Double hop works! Joe better known as “clymb3r” explains in details [here](http://www.powershellmagazine.com/2014/03/06/accidental-sabotage-beware-of-credssp/).  

![Double-Hop](https://threatpointer.blog/wp-content/uploads/2016/02/double-hop.png?w=576&h=203 align="left")

**Applications:** That being said on the network side, we may encounter products that may use Windows Remoting while integrating Microsoft Products. Examples, application integrating with IIS Server components or fetching alerts from the SCOM server etc. Both these applications from Microsoft provide PowerShell Providers for scripting purposes. It’s only a matter of time that we see this to become a common trend. For use of simplicity applications may simply accept user credentials and pass them to remote computers using Basic authentication. If something of this is noticed a Wireshark capture should help us identify such issues.

**Untrusted Domains:** When attempted to remote from one domain to another domain and there is no trust between them. There is no way for my computer to verify the identity to whatever that I connect to. My computer is going to send packets to an IP address but there is no way to verify if the IP belonged to the actual computer I intended to send. Simply put, I will not have mutual authentication. To fix it, we need to disable the need for mutual authentication & the need for trusts between domains.

SSL & Trusted host list are the two tools that could come handy in this case. For SSL we could either go to an external vendor to issue us certificates which is definitely going to cost us or we could use ADCS (Active Directory Certificate Services) & use this to provide

Trusted host list can be found here: Wsman:localhost\\client\\trusted host which is an item listed empty by default

**Issues- Common Credentials :** In some cases, we you may notice that Remoting into computers of the same domain and an untrusted domain with a user that is common on both might just work. What I mean by this is, when we specify a computer list that are part of different domains in the same script block it will not work. However, some administrators might have a common user that are part of both the computers (different domains) and not specifying a domain could just do the trick

Classic Example: Invoke-command -ScriptBlock {get-evenlog -LogName security -Newest 10 } -ComputerName pos.trusteddomain.com, pos1.Untrusteddomain.com -Credential Administrator

Not many a times we encounter this issue at our production sites but is something worth to check. The fix is pretty straightforward to have no common passwords. If the username and password is same on both the computers then it would work. However, at the same time if explicitly mentioned the domain name within the command, it would not work as it would use Kerberos authentication

**Issues- Trusted Host**: Normally we would add in the computer name to the list and would do the trick. However, if a network attacker were to spoof DNS on that IP Address, then there is no way for you to tell that you are actually connecting to the computer that you intended too. We are giving up on mutual authentication, we don’t know for a fact that sent credentials were to the real ‘computer.domain’.

However, this issues could be resolved when enabled we enable SSL. You will need generate a CSR for the computer within the untrusted domain and have it provided to the Certificate Authority. Depending on your choice you make, either an external one or an Internal Active Directory provided one. Please make sure that we are providing complete details while generating the Certificate Request. You may then import the Certificate issued to your computer and the Root Certificate if that requested from an external vendor. Once imported, grab the Thumbprint of the certificate issued to you and create your new HTTPS listener.

Command:

```plaintext
New-WSManlnstance -ResourceURI winrm/config/Listener -SelectorSet 
@{Address=’*”;Transport=’HTTPS’) -ValueSet @{Hostname=’computer.untrustedDomain.com’;
CertificateThumbprint=’60E49D63E304B9E1D148608346462713E8F1C16E’}
```

![cmd](https://threatpointer.blog/wp-content/uploads/2016/02/cmd.png?w=650&h=29 align="left")

Test the connection: Enter-PsSession -Computername computer.untrustedDomain.com -UseSSL -Credential untrustedDomain\\Adminsitrator

**AllowUnencrypted = True:** Quite often we see a lot of code snippets that are shared over the Internet and we simply use them within our environments. Once such example is blogs that show you how to enable CredSSP without discussing the problems around them. This is what we are not supposed to do. . 🙂

```plaintext
winrm set winrm/config/client/auth @{Basic="true"}
```

winrm set winrm/config/service/auth @{Basic=”true”}

winrm set winrm/config/service @{AllowUnencrypted=”true”}

Microsoft’s PowerShell Team explains the same in great details within their [blog](http://blogs.msdn.com/b/powershell/archive/2015/10/27/compromising-yourself-with-winrm-s-allowunencrypted-true.aspx): Compromising Yourself with WinRM’s “AllowUnencrypted = True”

**Delegated Administration**: The goal of this concept is to allow a certain user group to run certain set of Administrative commands. For example, enabling Help Desk to Enable or Disable Active Directory account only using this channel. We would create a New-PsSessionConfigurationFile, import the Modules needed for the task and provide the cmdlets needed. If I were to stick to the example I mentioned, the command would look something like this-

```plaintext
Command: New-PSSessionConfigurationFile -Path c:\helpdesk.pssc -ModuleToImport
```

```plaintext
activedirectory -VisableCmdlets 'Enable-AdAccount','Disable-ADAccount'
```

This would create a **.pssc** file which is nothing but a text file which will have plenty of parameters, just as you would find within an Hashtable- ‘key = value’. Then, the next step would be to register the endpoint. Following screenshot explains the same-

![Register-PsSessionConfig](https://threatpointer.blog/wp-content/uploads/2016/02/register-pssessionconfig.png?w=840&h=543 align="left")

Here we would simply add the HelpDesk User Group and we would we all okay.

**Issues: Delegated Admins:** Usually the SDDL parameter is not completely understood & end up providing User Groups that may not necessarily require permissions. One would simply Google how to look up the Security Descriptor for a user group and append it to the existing ones. I have found a lot of issues around this area, where all authenticated users would Read\\Write permissions and is something to watch out for.

**PowerShell Session Configuration:** The *Set-PSSessionConfiguration* cmdlet changes the properties of the session configurations on the local computer. Beginning in Windows PowerShell 3.0, you can use a session configuration file to define a session configuration. This feature provides a simple and discoverable method for setting and changing the properties of sessions that use the session configuration. Session configurations define the environment of remote sessions (PSSessions) that connect to the local computer. Every PSSession uses a session configuration. The session configuration determines the features of the PSSession, such as the modules that are available in the session, the cmdlets that are permitted to run, the language mode, quotas, and timeouts. The security descriptor (SDDL) of the session configuration determines who can use the session configuration to connect to the local computer.

A quick Get-PsSessionConfiguration gives you an exhaustive list of settings that are available. The screenshot listed below is a partial output of the command-

![Get-PsConfig](https://threatpointer.blog/wp-content/uploads/2016/02/get-psconfig.png?w=830&h=827 align="left")

We can watch out for some of the following settings to see if they are away from the default ones-

* Check for the ‘SecurityDescriptorSddl’ or the Permission parameter both should resolve to the same, explaining what user groups have permissions
    
* ‘RunAsPassword’ -is blank by default
    
* ‘RunAsUser’ – is blank by default
    

**PowerShell Web Remoting:** It gives you a web based interface into a gateway computer which then uses PowerShell Remoting to get you to a computer that you need. To be able to install: Add-WindowsFeature windowspowershellwebaccess, this will make module available called PowerShellWebAccess. Exploring this would give you Install-PswaWebApplication.

This will only work when HTTPS is enabled. This means the computer at which we are attempting to install should be enabled with SSL something which we have seen previously. You should see something like this when you bring this up on your Web Browser

![PsWebAccess](https://threatpointer.blog/wp-content/uploads/2016/02/pswebaccess.png?w=546&h=439 align="left")

**Issues Web Remoting:** The most common pit fall is when configuring the Add-PswaAuthoriztionRule. Microsoft’s documentation on this explains the classic problem [here](https://technet.microsoft.com/en-us/library/jj592890(v=wps.630).aspx). Example 6: use of wildcard entries enables all users to be able to connect to all computers for any configuration 🙂

By no means this is close to being an exhaustive list to perform checks when auditing Windows Remoting features. However, the aim was to set you out on the right direction so we explore further. If you have done something different or would like to share your experiences with Windows Remoting- hit me on twitter: @threatpointer or write me: mohammed.tanveer1@gmail.com. Hoping this was helpful to you guys, until next time, keep PowerShelling!

Cheers!

References:

* [https://msdn.microsoft.com/en-us/library/aa384470(v=vs.85).aspx](https://msdn.microsoft.com/en-us/library/aa384470(v=vs.85).aspx)
    
* [http://blogs.msdn.com/b/powershell/archive/2015/10/27/compromising-yourself-with-winrm-s-allowunencrypted-true.aspx](http://blogs.msdn.com/b/powershell/archive/2015/10/27/compromising-yourself-with-winrm-s-allowunencrypted-true.aspx)
    
* [https://www.penflip.com/powershellorg/secrets-of-powershell-remoting/blob/master/configuring-remoting-via-gpo.txt](https://www.penflip.com/powershellorg/secrets-of-powershell-remoting/blob/master/configuring-remoting-via-gpo.txt)
    
* [https://technet.microsoft.com/en-us/library/ee126186%28v=vs.85%29.aspx?f=255&MSPPError=-2147217396](https://technet.microsoft.com/en-us/library/ee126186%28v=vs.85%29.aspx?f=255&MSPPError=-2147217396)
    
* [http://www.powershellmagazine.com/2014/03/06/accidental-sabotage-beware-of-credssp/](http://www.powershellmagazine.com/2014/03/06/accidental-sabotage-beware-of-credssp/)