
# Table of contents

- [Table of contents](#table-of-contents)
  - [This tutorial will grant you following benefits](#this-tutorial-will-grant-you-following-benefits)
  - [Summary of the problem](#summary-of-the-problem)
  - [Tutorial steps](#tutorial-steps)
  - [Final step and your opinion](#final-step-and-your-opinion)
  - [Troubleshooting GPG key caching](#troubleshooting-gpg-key-caching)
  - [References](#references)

## This tutorial will grant you following benefits

>- work with git in Powershell, you won't need to use MSYS terminal any more
>- you will have colored prompt in powershell when in working tree
>- git will not ask you for ssh password every time (not even after reboot) because ssh-agent will run as windows service.
>- your commits will be automatically signed by default
>- git will use gpg-agent from gpg4win suite, to sign your commits (meaning being able to manage and generate your keys with Kleopatra as well as many other GUI options for GPG)
>- you will be asked for gpg key only once, with the ability to customize how often gpg asks you for GPG key password
>- you will communicate with github over ssh

## Summary of the problem

git comes as you might know bundled with it's own gpg and ssh executables, however if you want git to use gpg4win version of gpg
for signing commits and if you don't want to be prompted for ssh and gpg keys every time you push and commit and if in addition
you want all that in powershell then you are likely to get into a lot of trouble.

Other uses of this setup include use of git with custom ssh and gpg for what ever reason, or if you just want to be able to cache keys in a more efficient ways.

the biggest problem is that ssh that comes with git doesn't run ssh-agent automatically, posh-git doesn't start
ssh-agent properly at the time of writing, (maybe some new version in the future will), because there is a conflict between
openSSH that comes with windows and ssh that comes with git, so the alternative way is to use OpenSSH that comes bundled
with Windows 10.
For more information see [OpenSSH in Windows](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_overview)

However even that won't work perfectly because bundled ssh in Windows 10 is out of date and outputs annoying
signature warning when doing push (at least on Windows 1903), so we need to install manually latest version of OpenSSH!

note that even if posh-git manages to start ssh it still won't be perfect for our key-caching goal because ssh that comes with git
doesn't offer windows service!

I won't go into details too much as to providing technical details why default setup doesn't work,
because it would take a lot of explanation, and since you're reading this you might already know that!

## Tutorial steps

**NOTE:** Portion of the steps here are based on instructions from documentation on `github.com` but updated to reflect powershell usage,
and some stuff has been removed because it's irrelevant with our setup.

Also some of the content is based on `git-scm.com` instructions and many other sites, the point is that all this information is now assembled
here in one place for a complete tutorial.

At the end of this tutorial is a list of links to original documentation and references that describe the problem.

**NOTE:** All of the commands below are executed in powershell,
also these steps have been tested on Windows 10 only

1. First if you didn't already download and install [gpg4win](https://www.gpg4win.org/download.html) and [git](https://git-scm.com/)

2. Windows 10 only step: next we need to get rid of faulty openSSH that comes with windows:

   - log into Administrative account (if you're not Administrator)
   - press `Windows key + I` to open settings
   - type "Optional Feature" from the list
   - select "Manage optional features" to go to the Setting App.
   - click on "OpenSSH Client" and uninstall

3. There are multiple ways to install chocolatey as explained [here](https://chocolatey.org/docs/installation) we'll use `Install-Package` version and if that doesn't work then `nuget.exe` option because these seem to be the most practical options.

4. If you are running an older version of windows such as Windows 7 it's possible your powershell is pretty old
so let's first check powershell version, Open up powershell and type:

    ```powershell
    $PSVersionTable.PSVersion
    ```

    In order for all of the commands in this tutorial to work you will need at least Windows PowerShell 5.x or PowerShell Core 6.0.
    If for some reason you can't upgrade powershell this tutorial provides a subset of alternative commands that will work
    except `posh-git` module which requires powershell minimum version 5.

    To grab and install latest PowerShell core follow this link [Get PowerShell Core](https://github.com/PowerShell/PowerShell/releases)

    Otherwise if you need to update Windows PowerShell then grab zipped powershell installer as windows update, but with some restricted installation opportunities
    as explained on this link:
    [Installing Windows PowerShell](https://docs.microsoft.com/en-us/powershell/scripting/install/installing-windows-powershell?view=powershell-6)

5. Before you're able to run PowerShell scripts on your machine, you need to set your local ExecutionPolicy to RemoteSigned

    More about [PowerShell Scopes](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_scopes?view=powershell-6)
    More about [PowerShell ExecutionPolicy](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.security/set-executionpolicy?view=powershell-6)

6. Open up Powershell (Admin) and check current execution policy:

    ```powershell
    Get-ExecutionPolicy
    ```

    **NOTE:** PowerShell Core defaults to `RemoteSigned` while Windows PowerShell defaults to `Restricted`

    if it's not RemoteSigned change execution policy:

    ```powershell
    Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
    ```

7. Because we'll install some modules, you might want to set `NuGet` and `PSGallery` package sources as trusted, also you might want to do it for both Administrative and standard account (by opening respective PowerShell for each user).

    **NOTE:** commands for nuget.org (NuGet provider) in this step and steps 8 and 9 that follow may fail in PowerShell core, and the workaround is not worth spending time, if your Core is up to date you're fine!
    I'll let you know the alternative options, for now just skip "NuGet" steps if it doesn't work.

    To see package sources first run:

    ```powershell
    Get-PackageSource
    ```

    To set nuget.org as trusted run:

    ```powershell
    Set-PackageSource -Name nuget.org -Trusted
    ```

    More information about [NuGet provider](https://docs.microsoft.com/en-us/nuget/what-is-nuget)

    To set PSGallery as trusted run:

    ```powershell
    Set-PSRepository -Name PSGallery -InstallationPolicy Trusted
    ```

    More information about [PowerShell Gallery](https://docs.microsoft.com/en-us/powershell/gallery/overview)

8. Next logical step is to install/update prerequisites.

    Let's first check if we have required NuGet package provider installed:

    ```powershell
    Get-PackageProvider
    ```

    If NuGet is not listed (or to check if it's out of date) find out if it's possible to install/update it:

    ```powershell
    Find-PackageProvider -Name "NuGet" -AllVersions
    ```

    If NuGet is out of date or not installed run new Powershell (Admin) instance and install it (or update with `-Force` switch),

    ```powershell
    Install-PackageProvider -Name NuGet
    ```

    Restart administrative PowerShell and then we also need to update PowerShellGet version which is required for latest posh-git, this is possible to install only with `-Force` switch as explained [here](https://docs.microsoft.com/en-us/powershell/scripting/gallery/installing-psget?view=powershell-7), otherwise you should get an error:

    To see currently installed versions:

    ```powershell
    Get-Module -ListAvailable -Name PowerShellGet
    ```

    To install new version on top of that is preinstalled:

    ```powershell
    Install-Module -Name PowerShellGet -Force
    ```

    To update existing PowerShellGet: (the one you personally already installed with `Install-Module`)

    ```powershell
    Update-Module -Name PowerShellGet
    ```

    If you get an error such as "Unable to find module providers" in Windows PowerShell (in core it might not work anyway) restart administrative powershell for each of these 2 sets of commands before executing them,
    and make sure you use `-Force` with `Install-Module`

9. Next again restart admin powershell and search for available chocolatey versions:

    **NOTE:** command below may fail in PowerShell core, if so there is "option 2" for you.

    ```powershell
    Find-Package chocolatey -MinimumVersion 0.10.14`
    ```

    Install chocolatey by using either option 1 or option 2:\
    **NOTE:** You can of course build openssh from source and exclude nuget.exe and chocolatey but I prefer the easy way.
    if you wish to build it from source grab ported sources here: [PowerShell/openssh-portable](https://github.com/PowerShell/openssh-portable)\
    **NOTE:** We'll install [this openssh](https://chocolatey.org/packages/openssh) which is based on Microsoft fork on github, the difference is that this one will also give us the option to install windows service for ssh-agent.\
    To input more parameters see previous openssh link, we use `-pre` to get latest prerelease.

    **OPTION 1: Using Install-Package**
    To download chocolatey, replace `RequiredVersion` option\
    with version obtained from `Find-Package` above:

    ```powershell
    Install-Package -Name chocolatey -ProviderName NuGet -RequiredVersion x.x.x
    ```

    Next to install chocolatey do:
    `Initialize-Chocolatey`

    this should finish installation, if this does not work then:\
    **NOTE:** update `cd` command to match chocolatey version

    ```powershell
    cd "C:\Program Files\PackageManagement\NuGet\Packages\chocolatey.0.10.14\tools"
    `& .\chocolateyInstall.ps1
    ```

    Finally to install OpenSSH, restart administrative PowerShell and run:\
    `choco install openssh -pre --package-parameters="/SSHAgentFeature"`

    **OPTION 2: Using nuget.exe**
    This is alternative method to install chocolatey and OpenSSH:

    Get latest `nuget.exe` from [NuGet](https://www.nuget.org/downloads), put it somewhere for example `C:\tools`
    and add `C:\tools` (or what ever path you choose) to `PATH` environment variable.

    **NOTE:**  we need `nuget.exe` in `PATH` to install [chocolatey](https://chocolatey.org/), and we need chocolatey because it will let us install [openssh](https://www.openssh.com).\
    **NOTE:** if you wish to install chocolately somewhere else update first command, and ensure chosen path is in `%PATH%` environment variable.

    ```powershell
    cd C:\tools
    nuget install chocolatey
    cd choco*\tools
    & .\chocolateyInstall.ps1
    choco install openssh -pre --package-parameters="/SSHAgentFeature"
    ```

10. For option 1, `Install-Package` will download chocolatey into:\
`C:\Program Files\PackageManagement\NuGet\Packages\chocolatey.x.x.x`.\
For option2: nuget.exe will temporary download chocolatey to `C:\tools\chocolatey.x.x.x`.

    In both cases chocolately will be installed into `%PROGRAMDATA%\chocolatey` and OpenSSH with service configuration will be installed into `C:\Program Files\OpenSSH-Win64`.

11. now open `services.msc` as Administrator and make sure `ssh-agent` service is started and set to automatic startup.

12. close Admin powershell and start new powershell instance (as standard user, if your acc isn't admin of course)
Please first ensure that ssh and gpg paths below are correct paths on your system, and if not update as needed, for example,
if you installed gpg4win or openSSH somewhere else.
You can verify installation path by typing:

    ```powershell
    where.exe ssh
    where.exe gpg
    ```

    Now start telling git about our setup: (btw. hard to tell why this silly syntax for paths)

    ```powershell
    git config --global --replace-all core.sshCommand "'C:\Program Files\OpenSSH-Win64\ssh.exe'"
    git config --global --replace-all gpg.program "C:\Program Files (x86)\GnuPG\bin\gpg.exe"
    ```

    And to verify these commands are input correctly:

    ```powershell
    git config --global --edit
    ```

    The input must look like this (I'm showing this so that tutorial changes don't mess up with expected result):

    ```none
    [core]
        sshCommand = 'C:\\Program Files\\OpenSSH-Win64\\ssh.exe'
    [gpg]
        program = C:\\Program Files (x86)\\GnuPG\\bin\\gpg.exe
    ```

    **NOTE:** above paths are pointing git to use gpg4win executables of gpg and our fresh installed openSSH portable as ssh program

13. Now everything is set up, and it's time to setup the keys!

    If you have existing gpg keys made by gpg4win in `%appdata%\gnupg` which you would like to backup first (because we might override
    them) then issue the following command:

    ```powershell
    robocopy $env:APPDATA\gnupg $env:APPDATA\gnupg-backup /MOVE /E
    ```

    if you have existing gpg keys in `~\.gnupg` which is where gpg keys are put by git version of gpg and you would like to use
    them with gpg4win to sign commits, then issue following commands to move entry conents into `%appdata%\gnupg`

    ```powershell
    robocopy $env:USERPROFILE\.gnupg $env:APPDATA\gnupg /MOVE /E
    ```

    If you have existing key somewhere else (ex. backup on USB) then import them with Kleopatra.

14. Otherwise if you don't have any gpg keys at all then generate new keys.
you can generate gpg keys with Kleopatra that comes with gpg4win or you can do the same in powershell by typing:

    ```powershell
    gpg --default-new-key-algo rsa4096 --gen-key
    ```

    At the prompt, specify your options, email, password etc. when asked.

15. Now it's time to gather information and tell git about our gpg keys:

    ```powershell
    gpg --list-secret-keys --keyid-format LONG
    ```

    From the list of GPG keys, copy the GPG key ID you'd like to use. In the following example, the GPG key ID is `3AA5C34371567BD2`

    example:

    ```powershell
    C:\Users\User> gpg --list-secret-keys --keyid-format LONG
    C:/Users/User/AppData/Roaming/gnupg/pubring.kbx
    -----------------------------------------------
    sec   4096R/3AA5C34371567BD2 2016-03-10 [expires: 2017-03-10]
        4096R/42B317FD4BA89E7A
    uid                 [ultimate] your_username (key for github) <youremail@mail.com>
    ssb   4096R/42B317FD4BA89E7A 2017-09-13 [E] [expires: 2017-09-12]
    ```

    Paste the text below, substituting in the GPG key ID you'd like to use.
    Tell git about your signing key, and tell it that you want to sign all commits by default:

    ```powershell
    git config --global user.signingkey 3AA5C34371567BD2
    git config --global commit.gpgsign true
    ```

    Export public key to clipboard (Note that Set-Clipboard cmdlet works only with newer powershell versions)
    if Set-Clipboard doesn't work export the the key using cmd, as shown in the second command:

    ```powershell
    gpg --armor --export 3AA5C34371567BD2 | Set-Clipboard
    ```

    Version to export public key to clipboard if `Set-Clipboard` doesn't work:

    ```powershell
    gpg --armor --export 3AA5C34371567BD2 > gpgkey.txt
    cmd
    clip < gpgkey.txt
    exit
    del gpgkey.txt
    ```

    Now go to your **Github account > settings > SSH and GPG Keys > Add new GPG key** and pres **CTRL+V** into box to paste the key.

16. same applies to ssh keys, if you don't have existing ones generate new ones, you don't need to worry about where keys
are (will be) stored because all ssh programs write keys to same directory that is `~\.ssh`

    type following to generate new keys:

    ```powershell
    ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
    ```

    When you're prompted to "Enter a file in which to save the key," press Enter.
    This accepts the default file location, also specify new password for the key.

    If your ssh keys are somewhere else, (ex. backed up on USB), you'll first need to create a new folder for them:\
    **NOTE:** I provide command for this because on server platforms it might not be possible with mouse.

    ```powershell
    mkdir $env:USERPROFILE\.ssh
    ```

    copy/paste `id_rsa` and `id_rsa.pub` from your backup into new `~\.ssh` folder.

    Once you have your keys in default location, add your private key to ssh-agent:

    ```powershell
    ssh-add $env:USERPROFILE\.ssh\id_rsa
    ```

    **NOTE:** `ssh-add` without arguments adds the default keys `~/.ssh/id_rsa, ~/.ssh/id_dsa, ~/.ssh/id_ecdsa. ~/ssh/id_ed25519, and ~/.ssh/identity` if they exist.

    Copy ssh public key to clipboard (again if `Set-Clipboard` doesn't work see second command below)

    ```powershell
    Get-Content $env:USERPROFILE\.ssh\id_rsa.pub | Set-Clipboard
    ```

    Version to copy ssh public key to clipboard if `Set-Clipboard` doesn't work:

    ```powershell
    Get-Content $env:USERPROFILE\.ssh\id_rsa.pub > sshkey.txt
    cmd
    clip < sshkey.txt
    exit
    del sshkey.txt
    ```

    Now go to your **Github account > settings > SSH and GPG Keys > Add new SSH key** and pres **CTRL+V** into box to paste the key.

17. You also want to tell git about yourself, for users of private email type in private email below ex. `random_string@users.noreply.github.com`:

    ```powershell
    git config --global user.name "your name or username"
    git config --global user.email youremail@example.com
    ```

18. next thing to do is to make `gpg-agent` cache your password longer, say for at least 2h so that you don't have to input
gpg key password every time you commit!

    Next we'll create `gpg-agent.conf` in `%appdata%\gnupg` and input settings into a file:

    ```powershell
    New-Item -Path $env:appdata\gnupg\gpg-agent.conf -ItemType File
    Set-Content -Path $env:appdata\gnupg\gpg-agent.conf -Encoding utf8 -Value "default-cache-ttl 7200"
    Add-Content -Path $env:appdata\gnupg\gpg-agent.conf -Encoding utf8 -Value "max-cache-ttl 14400"
    ```

    To set pin entry to wait 2 minutes for input set:

    ```powershell
    Add-Content -Path $env:appdata\gnupg\gpg-agent.conf -Encoding utf8 -Value "pinentry-timeout 120"
    ```

    To have gpg-agent log it's activity about password caching run this:

    ```powershell
    Add-Content -Path $env:appdata\gnupg\gpg-agent.conf -Encoding utf8 -Value "debug-level guru"
    Add-Content -Path $env:appdata\gnupg\gpg-agent.conf -Encoding utf8 -Value "log-file $env:appdata\gnupg\gpg-agent.log"
    ```

    Next we stop gpg-agent and start it again:

    ```powershell
    gpgconf --kill gpg-agent
    gpgconf --launch gpg-agent
    ```

    you can adjust these numbers which represent for how many seconds gpg-agent will cache password.\
    Above numbers mean, default-cache 2h, max-cache 4h and pin entry 2 minutes.

    The meaning of these options is as follows:

    - default-cache-ttl n
  
        Sets the time a cache entry is valid to n seconds. Each time a cache entry is accessed, the entry's timer is reset.
        Default is 10 minutes (600 seconds)

    - max-cache-ttl n
  
        Sets the maximum time a cache entry is valid to n seconds. After this time a cache entry will be expired even if it has been accessed recently.
        Default is 2 hours (7200 seconds)

    - pinentry-timeout n

        This option sets the pinentry to timeout after n seconds with no user input.
        Default value of 0 does not ask the pinentry to timeout

19. now last thing to do is to install posh-git module so that you get color prompt in powershell when working with git repos.

    More about [posh-git](https://github.com/dahlbyk/posh-git) and it's options

    **NOTE:** If you have at least PowerShell 5 with PackageManagement installed, you can use the package manager to install posh-git for you.

    Close Administrative powershell and the other powershell too, to update environment and open a new user mode powershell, then install posh-git:

    First we need to check and if necessary set execution policy for current user,
    Open up Powershell (standard user) and check current execution policy:

    ```powershell
    Get-ExecutionPolicy
    ```

    if it's not RemoteSigned change execution policy:

    ```powershell
    Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
    ```

    If you never installed posh-git then:

    ```powershell
    PowerShellGet\Install-Module posh-git -Scope CurrentUser -AllowPrerelease
    ```

    Otherwise to update existing one:

    ```powershell
    PowerShellGet\Update-Module posh-git
    ```

    Finally import module and tell powershell to import it each next time:

    ```powershell
    Import-Module posh-git
    Add-PoshGitToProfile -AllHosts
    ```

    **NOTE:** Keep in mind, that there are multiple $profile scripts. ex: one for the console and a separate one for the ISE.
    that's why we use `-AllHosts`

20. Final step is to reverse/reset execution policy for both administrative and user mode PowerShell:\
    For Windows PowerShell, default its `Restricted` and for PowerShell Core it is `RemoteSigned`

    ```powershell
    Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
    ```

## Final step and your opinion

That's it, it's recommended to reboot system and start testing your git setup with PowerShell, try experimenting push/pull/commit etc.., but basically you should now be able to work with git in powershell, and the ssh-agent will not ask you for password because it runs as service on Windows, and also gpg-agent will ask you for password either
every 2h or max. 4h

**NOTE:** Now Treat your MSYS terminal that comes with git as obsolete on your system, you have no reason to use it anymore, because even though git will use your new setup the MSYS terminal it self continues to use old settings and variables and ssh-agent will most likely not cache your keys because inside MSYS terminal environment there are two ssh programs resulting in conflict.

If you encounter problems or have suggestion please report.
happy coding! :grinning:

## Troubleshooting GPG key caching

- Show options used by gpg-agent now

    `gpgconf --list-options gpg-agent`

    In this output you want to see values your options only and make sure values are those you entered into gpg-agent.conf

- See if gpg-agent has issues with options

    `gpgconf --check-options gpg-agent`

    result of `gpg-agent.exe:1:1:` means no problems, anything else is error

- List directories used by gpg suite

    `gpgconf --list-dirs`

    what you are looking for here is `homedir` which is where your gpg-agent.conf must be saved to be recognized

- Test gpg-agent.conf file (replace USERNAME with your )

    `gpgconf --check-config "$env:appdata\\gnupg\\gpg-agent.conf"`

    Test gpg-agent.conf file, this will likely fail probably a bug

- Test gpg software suite

    `gpgconf --check-programs`
    Verify all programs of gpg suite are OK (status must be :1:1)

- parse the configuration file and returns with failure if the configuration file would prevent gpg from startup

    `gpg-agent --gpgconf-test`

- If things don't work well wit gpg even after reboot, make sure to **delete** `gpg-agent.conf` file, and set these settings in Kleopatra
  program that comes with gpg4win in following location

  - Kleopatra -> Settings -> Configure Kleopatra -> GnuPG system -> Private keys -> Options controlling the security
  - Here setup caching options, click apply/OK to let Kleopatra generate new gpg-agent.conf and reboot system

## References

This were the most useful links to help me out assemble this tutorial:

- [Connecting to GitHub with SSH](https://docs.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh)

- [Git in PowerShell](https://git-scm.com/book/en/v2/Appendix-A%3A-Git-in-Other-Environments-Git-in-PowerShell)

- [Windows 10 version 1803 broke my git SSH](https://adamralph.com/2018/05/15/windows-10-version-1803-broke-my-git-ssh/)

- [Moving from Windows 1809's OpenSSH to OpenSSH Portable](https://blog.frankfu.com.au/2019/03/21/moving-from-windows-1809s-openssh-to-openssh-portable)

- [GPG Agent Configuration](https://www.gnupg.org/documentation/manuals/gnupg/Agent-Options.html)

- [Invoking gpgconf](https://www.gnupg.org/documentation/manuals/gnupg/Invoking-gpgconf.html)

- [GPG Esoteric Options](https://www.gnupg.org/documentation/manuals/gnupg/GPG-Esoteric-Options.html)
