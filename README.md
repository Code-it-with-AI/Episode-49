# Episode-49: Remote to Local LLM

Carl shows how to remotely access your local machine and run Copilot or Claude against a local LLM

📺 YouTube video: https://youtu.be/QNwG59oBvac

🏠 Code it with AI Home Page: [https://codeitwithai.com](https://codeitwithai.com/)

This guide explains how to securely use an iPhone to connect to a Windows development PC, launch Claude Code or GitHub Copilot CLI on that PC, and optionally continue using a local LLM running on the Windows machine. It also covers transferring screenshots from the iPhone into the remote AI coding session.

The approach uses Windows OpenSSH Server, Secure ShellFish on iOS, and Tailscale for secure access when away from the local network. It does not require exposing SSH directly to the public Internet.

## What You Are Building

The finished setup looks like this:

``` text
iPhone
  |
  +-- Secure ShellFish (SSH/SFTP client)
  |
  +-- Tailscale
          |
          | Encrypted Tailscale network
          |
Windows development PC
  |
  +-- Tailscale
  |
  +-- Windows OpenSSH Server (SSH/SFTP)
  |
  +-- Claude Code
  |
  +-- GitHub Copilot CLI
  |
  +-- Optional local LLM server
```

Claude Code and Copilot CLI actually run on the Windows PC. The iPhone is simply a remote terminal. This is especially useful if your coding agent is configured to use a local LLM that is only accessible from the development PC.

## 1. Install Windows OpenSSH Server

Open **PowerShell as Administrator** and install Microsoft's built-in OpenSSH Server:

``` powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

Start the SSH server and configure it to start automatically with Windows:

``` powershell
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

Verify that the SSH service is running:

``` powershell
Get-Service sshd
```

Windows normally creates the OpenSSH firewall rule during installation. Verify it with:

``` powershell
Get-NetFirewallRule -Name OpenSSH-Server-In-TCP |
    Select-Object Name, Enabled, Direction, Action
```

You want an enabled inbound `Allow` rule for OpenSSH.

## 2. Find the Windows PC's Local IP Address

While initially configuring everything on your local network, find the Windows PC's IPv4 address:

``` powershell
ipconfig
```

Look for the IPv4 address of the Ethernet or Wi-Fi adapter that connects the PC to your network. It will typically look something like:

``` text
192.168.1.50
```

Substitute your own local address throughout this guide.

Your iPhone and Windows PC do not have to use the same physical connection type. For example, the Windows PC can be connected by Ethernet while the iPhone is connected by Wi-Fi. They only need to be able to communicate through the same local network.

## 3. Install an SSH Client on the iPhone

Secure ShellFish works particularly well for this setup because it supports both interactive SSH terminals and SFTP file transfers, including a convenient clipboard-image upload workflow.

When you first run ShellFish, tell it that you want to connect to a Windows server.

Create an SSH connection using your Windows PC's local address:

``` text
Host:      192.168.1.50
Port:      22
Username:  your-windows-username
```

Replace the example address and username with your own.

Test basic SSH connectivity before worrying about remote Internet access or image transfers.

## 4. Configure SSH Key Authentication

Generate an SSH key in your iPhone SSH client and copy its public key.

For a normal Windows account, OpenSSH commonly uses the user's `.ssh\authorized_keys` file. There is an important Windows-specific exception: if the Windows account belongs to the local Administrators group, Windows OpenSSH normally uses this shared file instead:

``` text
C:\ProgramData\ssh\administrators_authorized_keys
```

Paste the iPhone client's public key into that file, one key per line.

### Notepad Gotcha

If you create the file with Notepad, make sure Notepad does not silently add a `.txt` extension.

The filename must be exactly:

``` text
administrators_authorized_keys
```

not:

``` text
administrators_authorized_keys.txt
```

You can check the file with:

``` powershell
Get-Content C:\ProgramData\ssh\administrators_authorized_keys
```

## 5. Fix the Authorized-Keys Permissions

Windows OpenSSH is strict about the permissions on `administrators_authorized_keys`.

Run these commands from an elevated PowerShell prompt:

``` powershell
icacls "C:\ProgramData\ssh\administrators_authorized_keys" /inheritance:r

icacls "C:\ProgramData\ssh\administrators_authorized_keys" /grant "Administrators:F"

icacls "C:\ProgramData\ssh\administrators_authorized_keys" /grant "SYSTEM:F"
```

Verify the result:

``` powershell
icacls "C:\ProgramData\ssh\administrators_authorized_keys"
```

You should see permissions similar to:

``` text
NT AUTHORITY\SYSTEM:(F)
BUILTIN\Administrators:(F)
```

If public-key authentication mysteriously fails even though the key itself looks correct, check these permissions before doing anything more complicated.

## 6. Windows Password Authentication Gotcha

Some SSH/SFTP operations or client configurations may require a Windows password even if you primarily intend to use public-key authentication.

You can inspect the local Windows account with:

``` powershell
Get-LocalUser -Name your-windows-username |
    Select-Object Name, PrincipalSource, PasswordRequired, PasswordLastSet
```

If the account uses a Microsoft account, `PrincipalSource` may show `MicrosoftAccount`.

Make sure the Windows account has a valid password that you know. A Windows Hello PIN is not necessarily interchangeable with the actual account password for SSH authentication.

## 7. Test the Remote Windows Shell

Once SSH authentication succeeds, you should get a normal Windows command prompt on the iPhone.

For example:

``` text
C:\Users\yourname>
```

At this point the iPhone is effectively operating a terminal on the development PC.

You can now launch Claude Code:

``` text
claude
```

or launch GitHub Copilot CLI using whatever command you normally use on that PC.

The coding agent is still executing on the Windows machine, so it retains access to the source tree, development tools, environment variables, local services, and any local LLM endpoint that the Windows machine can reach.

This is an important distinction from a cloud-hosted remote-agent feature. If your AI coding tool is configured to use a local model, SSH lets you remotely operate the same local process instead of moving the agent into the cloud.

## 8. Using Full-Screen Terminal Interfaces on an iPhone

Claude Code and similar terminal applications sometimes display menus that expect arrow keys, Escape, Tab, or other keys that are inconvenient on the normal iPhone keyboard.

Secure ShellFish provides terminal accessory controls for these keys. Use those controls instead of assuming the standard iPhone keyboard will expose a conventional desktop arrow-key layout.

This matters during prompts such as trusting a repository or selecting an option from a terminal menu.

## 9. Enable and Verify SFTP

SSH gives you the terminal. SFTP is what makes the image-transfer workflow useful.

Open:

``` text
C:\ProgramData\ssh\sshd_config
```

and locate the SFTP subsystem configuration.

A typical configuration may look like:

``` text
Subsystem sftp sftp-server.exe
```

Verify that the Windows SFTP server executable exists:

``` powershell
Get-Item C:\Windows\System32\OpenSSH\sftp-server.exe
```

A more explicit configuration, and the one used successfully during this setup, is:

``` text
Subsystem sftp C:/Windows/System32/OpenSSH/sftp-server.exe
```

Using the complete path avoids depending on the environment used when `sshd` launches the SFTP subsystem.

Note the forward slashes in the `sshd_config` path.

### Comments in sshd_config

A line beginning with `#` is a comment and is not active.

For example:

``` text
#Subsystem sftp sftp-server.exe
```

does not configure the SFTP subsystem.

## 10. Validate sshd_config Before Restarting SSH

After changing `sshd_config`, validate it before restarting the service:

``` powershell
C:\Windows\System32\OpenSSH\sshd.exe -t
```

If the command returns to the prompt with no output, the configuration passed validation.

Then restart OpenSSH:

``` powershell
Restart-Service sshd -Force
```

The `-Force` option may be necessary because Windows can report that `sshd` has dependent services.

Confirm that the service came back:

``` powershell
Get-Service sshd
```

Restarting `sshd` terminates existing SSH connections, so reconnect from the iPhone afterward.

## 11. Upload Screenshots Directly from the iPhone

This is one of the biggest advantages of using Secure ShellFish for an AI coding workflow.

Instead of manually saving a screenshot, opening an SFTP application, navigating to a directory, uploading the file, returning to the terminal, and typing the filename, you can use the clipboard.

The workflow is:

1.  Take a screenshot or open an existing image on the iPhone.
2.  Copy the image to the clipboard.
3.  Return to the ShellFish terminal.
4.  Paste.
5.  ShellFish detects that the clipboard contains an image and uploads it through SFTP.
6.  An upload progress indicator appears.
7.  After the upload, ShellFish presents a dialog that lets you insert the remote file path into the terminal.
8.  Insert the path into the Claude Code or Copilot prompt.
9.  Tell the coding agent to inspect the image.

ShellFish may create a filename resembling:

``` text
pasted-from-shellfish-20260930-192307.png
```

The important point is that the image itself is transferred to the Windows PC. Claude or Copilot then reads the local file from the Windows filesystem.

## 12. Understand the Separate SSH and SFTP Paths

A useful troubleshooting lesson is that an interactive SSH terminal and an SFTP transfer can behave differently.

You can have a perfectly responsive terminal while an image upload fails.

When debugging, think of the process as separate layers:

``` text
SSH connection
    |
Interactive terminal
```

and:

``` text
SSH connection
    |
SFTP subsystem
    |
Image/file upload
    |
Remote filename
    |
Claude/Copilot reads file
```

If the terminal still responds but an image upload fails, do not immediately assume the entire SSH connection or VPN has failed.

## 13. Turn On OpenSSH Logging When Things Go Wrong

Windows OpenSSH logging was extremely useful in diagnosing SFTP problems.

Logs can be configured to appear under:

``` text
C:\ProgramData\ssh\logs
```

The main server log is typically:

``` text
C:\ProgramData\ssh\logs\sshd.log
```

To inspect recent activity:

``` powershell
Get-Content C:\ProgramData\ssh\logs\sshd.log -Tail 250
```

At a sufficiently detailed logging level, you can see the iPhone client connect, authenticate, request an SFTP subsystem, and disconnect.

One useful diagnostic encountered during setup was:

``` text
subsystem request for sftp by user <username>
subsystem: cannot stat sftp-server.exe: No such file or directory
subsystem: exec() sftp-server.exe
```

That led to explicitly configuring:

``` text
Subsystem sftp C:/Windows/System32/OpenSSH/sftp-server.exe
```

If normal SSH works while SFTP behaves strangely, the OpenSSH server log is one of the first places worth checking.

## 14. Do Not Expose Port 22 to the Internet Unless You Have a Good Reason

Everything described so far works while the iPhone and Windows PC are on the same local network.

The next challenge is using the setup from a gym, hotel, coffee shop, cellular connection, or anywhere else outside the local network.

A tempting solution is to configure the router to forward a public Internet port to TCP port 22 on the Windows PC.

That is unnecessary for this setup and exposes the SSH service directly to Internet scanning and login attempts.

A much cleaner solution is Tailscale.

## 15. Install Tailscale on Windows

Install Tailscale on the Windows development PC. One convenient method is:

``` powershell
winget install --id Tailscale.Tailscale -e
```

Launch Tailscale and authenticate. If you want a free account, create it using a generic gmail.com address or your Apple account. I used my Apple account. I also chose that it was for personal use.

Tailscale creates a private encrypted network between your devices. Each device receives a Tailscale IP address, typically in the `100.x.x.x` range.

Check the Windows PC with:

``` powershell
tailscale status
```

You might see something like:

``` text
100.100.20.30  my-development-pc  account@  windows  -
```

Your address will be different. Substitute your own Tailscale address anywhere this guide shows an example.

## 16. Tailscale Account/Plan Gotcha

If you previously used Tailscale with a company or custom-domain identity, you may discover that the old tailnet is associated with a business trial or paid plan.

For a personal setup, it may be simpler to create a separate personal tailnet using a personal identity provider such as Apple.

In the setup documented here, an older custom-domain tailnet was left alone and a new free personal tailnet was created with Sign in with Apple.

If Apple's Hide My Email feature is enabled, the Tailscale account may display an Apple private-relay email address. That is normal.

The important thing is that the Windows PC and iPhone must ultimately be logged into the same tailnet.

## 17. Install Tailscale on the iPhone

Install Tailscale on the iPhone and authenticate using the same tailnet as the Windows PC.

iOS will ask permission to install or activate a VPN configuration. Allow it. Tailscale uses that network interface to create the encrypted connection.

Once connected, the Tailscale application should show both the iPhone and Windows PC.

For example:

``` text
iphone                100.70.40.10
my-development-pc     100.100.20.30
```

Again, use your own addresses.

## 18. Point ShellFish at the Tailscale Address

Once both devices are on Tailscale, change the ShellFish SSH connection from the Windows PC's LAN address:

``` text
192.168.1.50
```

to its Tailscale address:

``` text
100.100.20.30
```

Keep the other settings the same:

``` text
Port:      22
Username:  your-windows-username
```

Do not copy the example IP addresses from this guide. Use the address shown by `tailscale status` on your own Windows machine.

A good test is to turn Wi-Fi off on the iPhone and connect over cellular. If the ShellFish terminal reaches the Windows PC, remote SSH through Tailscale is working.

No router port forwarding or public SSH endpoint is required.

## 19. A Cellular/Tailscale File-Transfer Gotcha

One issue remained during testing and is worth documenting rather than pretending the setup was flawless.

Interactive SSH over Tailscale and cellular worked well. Claude Code and Copilot CLI could be operated remotely.

However, repeated screenshot/SFTP transfers through ShellFish became unreliable while the iPhone was using cellular through Tailscale. The terminal itself remained responsive.

When the iPhone returned to the local Wi-Fi network, screenshot uploads became smooth again.

That means the known-good state was:

``` text
iPhone on local Wi-Fi
    -> ShellFish SSH/SFTP
    -> Windows PC
    -> Claude/Copilot
```

and:

``` text
iPhone on cellular
    -> Tailscale
    -> ShellFish SSH
    -> Windows PC
    -> Claude/Copilot
```

The part that still warranted additional investigation was:

``` text
iPhone on cellular
    -> Tailscale
    -> ShellFish SFTP/image uploads
    -> Windows PC
```

If you encounter the same behavior, first verify whether ordinary terminal commands still work. If they do, troubleshoot the SFTP/file-transfer path separately rather than rebuilding your SSH or Tailscale configuration.

## 20. Useful Diagnostic Commands

Check Tailscale:

``` powershell
tailscale status
```

Check OpenSSH:

``` powershell
Get-Service sshd
```

Restart OpenSSH:

``` powershell
Restart-Service sshd -Force
```

Validate `sshd_config`:

``` powershell
C:\Windows\System32\OpenSSH\sshd.exe -t
```

Check the configured SFTP subsystem:

``` powershell
Select-String -Path C:\ProgramData\ssh\sshd_config -Pattern "sftp"
```

Verify the SFTP executable:

``` powershell
Get-Item C:\Windows\System32\OpenSSH\sftp-server.exe
```

Inspect recent OpenSSH logging:

``` powershell
Get-Content C:\ProgramData\ssh\logs\sshd.log -Tail 250
```

Inspect administrator authorized keys:

``` powershell
Get-Content C:\ProgramData\ssh\administrators_authorized_keys
```

Check authorized-key permissions:

``` powershell
icacls C:\ProgramData\ssh\administrators_authorized_keys
```

Check the Windows network profile:

``` powershell
Get-NetConnectionProfile
```

Check the Windows account:

``` powershell
Get-LocalUser -Name your-windows-username |
    Select-Object Name, PrincipalSource, PasswordRequired, PasswordLastSet
```

## 21. Final Result

With this setup, an iPhone can act as a practical remote front end for a Windows AI development workstation.

The Windows machine continues doing the actual work. Claude Code and GitHub Copilot CLI run exactly where they normally run, with access to the repository, compilers, development tools, and local services. If the coding agent uses a local LLM, that model remains on the development machine as well.

Secure ShellFish provides the terminal and convenient screenshot transfer workflow. Windows OpenSSH provides SSH and SFTP. Tailscale provides secure remote connectivity without exposing the SSH server directly to the public Internet.

The result is particularly useful when you want to check on an agent, answer a prompt, give it another task, or send it a screenshot while away from your desk---all from an iPhone.
