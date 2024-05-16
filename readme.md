# External Dynamic Lists

This repository contains External Dynamic Lists (known as EDLs) - text files referenced
from Gen2 web-filtering policy (currently deployed at 4WTC only).

## Repository structure:

- ```content/edl/prod``` - Production EDLs
- ```content/edl/lab```  - 4WTC Lab EDLs

- ```content/nac```       - HTML pages used by NAC
- ```content/isolation``` - Landing page used by Gen2 isolation feature

---
# Description of production EDLs

### Break-glass
These EDLs allow you to completely bypass
firewall security policy. They should be used
**_only_ as a temporary workaround** for IPS/URL/App-ID 
false-positives or to tackle an unexpected behaviour 
of a security policy rule.

- ```ip-src-break-glass.txt``` - IP-source based
- ```ip-dst-break-glass.txt``` - IP-destination based
- ```url-break-glass.txt```    - URL-based

Changes to the IP-source based break-glass EDL requires approval by a senior team member. Current set of approvers can be found in the CODEOWNERS file in the root of the repository.

---
### TLS/SSL-decryption

These EDLs allow you to exclude a source or a destination from decryption: 

- ```ip-dst-no-decrypt.txt``` - connections to these IP-addresses will never be decrypted
- ```ip-src-no-decrypt.txt``` - connections from these IP-addresses will never be decrypted
- ```url-no-decrypt.txt```    - HTTPS connections to these URLs will never be decrypted

The EDL below allows to address potential HTTP/2 compatibility issues and to allow decryption 
of websites that support only legacy encryption:

- ```url-http2-downgrade.txt``` - HTTP/2 connections to these URLs will
be downgraded to HTTPv1.1, and legacy protocols and algorithms (SSLv3.0, MD5 etc.) will be allowed  

---
### Unconditional block

EDLs in this section allow you to unconditionally block network connections.
The main use case for these lists is **Incident Response** when an internal
host needs to be isolated from the Internet, or when a remote 
host must be denied access to a KKR public resource. Similarly, connectivity may need to 
be blocked based on a destination IP address or URL.

- ```ip-dst-full-block.txt``` - ALL connections to these IP-addresses will be blocked 
- ```ip-src-full-block.txt``` - ALL connections from these IP-addresses will be blocked
- ```url-full-block.dst```    - ALL connections to these URLs will be blocked

When blocking a malicious HTTP(S)-based destinations the URL-based EDL is likely to be preferable 
over the IP-based one. In some cases it would make sense to block both IP address and URL - seek
guidance from the Threat Intel team when in doubt.

---
### Other EDLs

- ```ip-dst-ssh-sftp-servers.txt```         - External resources reachable via SSH/SFTP 
- ```ip-dst-ssh-sftp-clients.txt```         - Internal end-points allowed to connect to the external resorces referenced in the EDL above (if a client is authenticated user then he/she must be added to the group `UG-ssh-sftp-clients` instead of this EDL)
- ```url-restricted-file-download.dst```    - External resources where members of the AD-group `UG-restricted-file-download` are allow to download executables from


---
# EDL syntax

When you create new entries in the EDLs you must follow the syntax described below.
## IP-based lists
The external dynamic list can include individual IP addresses, subnet addresses (address/mask), 
or range of IP addresses. In addition, the block list can include comments and special 
characters such as ```*``` , ```:``` , ```;``` , ```#```, or ```/```. 

The syntax for each line in the list is:

```[address OR address/mask OR start_address-end_address] [space] [comment]```

Enter each IP address/range/subnet in a new line; 
URLs or domains are not supported in this list. A subnet or an IP address range, 
such as ```92.168.20.0/24``` or ```192.168.20.40-192.168.20.50```, 
count as one IP address entry and not as multiple IP addresses (in the context of capacity planing). 

You should add comments when possible, the comment must be on the same line as the IP address/range/subnet. 
The space at the end of the IP address is the delimiter that separates a comment from the 
IP address.

There are no strict rules about content of the comments, but they **must be** concise and **clear
to yourself** because you will be asked to explain why the entry was made (see references to regular review cycles 
in the description of the EDLs above). For longer and more descriptive comments you can leverage Git/GitHub 
comment field when you commit a change.

Examples:
```commandline
192.168.20.10/32 John Doe issues with AD GPO preventing installation of root CA :: Zia :: 28 Mar 2022  
192.168.20.0/24 test internal subnet; will remove shortly :: Andrew ::  6 Apr 2022
192.168.20.40-192.168.20.50  
```
The last entry in the example above is valid, but it would be a poor practice to make it like this, 
because the entry does not contain a comment. Thus, SecOps would need to review Git commit 
history and comments, and the person who created the entry may struggle to remember
why it was made. COMMENT YOUR CHANGES.

## URL-based lists

### Basic principles

- Do **not** include URL protocol (http/https)
- Each entry can be up to 255 character long
- Entries are **not** case-sensitive
- Enter exact URL (_try to stick only to the host and domain part of the URL_) **OR** 
a wildcard (_see next the next section for details_)
- In-line **comments are not supported** (leverage Git/GitHub comments field instead)
- In principle an IP-address can be specified in the URL EDL but this would be really confusing
and be considered a poor practice. The firewall would interpret such entry as a plain URL and would be looking 
for it in the HTTP Host header or TLS SNI field, but **not** in IP header - bear this in mind! 

### Wildcard guidelines

- Each URL is made up of _tokens_ - substrings separated by these symbols: ```.``` 
```/``` ```?``` ```&``` ```=``` ```;``` ```+```.
- A token can be replaced by a wildcard symbol - either ```*``` or ```^```
- ```^``` would match only one token
- ```*``` would match one **or more** token(s)
- If you use a wildcard symbol there must be no other characters in the same token. 
For example, the url ```www.kk*.com``` would be **invalid** because there are two other 
characters (```kk```) in the same token where the wildcard symbol ```*``` is used.
- Unless you use a trailing ```/``` there will be implicit wildcard ```*``` implying multiple 
tokens. Thus, if you specify ```*.kkr.com``` it won't only match ```www.kkr.com``` or ```www.something.kkr.com``` 
but also ```www.kkr.com.malware.com``` 

Examples:
- `google.com` will match `google.com`, `google.co.ua` and `google.com.randomwebsite.com`
- `google.com/` will match only `google.com`
- `*.google.com` will match `blog1.blog2.google.com.au.us` and will also match `blog1.google.com`
- `^.google.com/` will match only `blog.google.com` but will not match `google.com` or `other.blog.google.com`
- `google.^.au/` will match only `google.com.au` and `google.uk.au` but will not match `google.com` or `google.com.au.website.info`
- `*.google.com/` will match `blog1.blog2.google.com` but will not match `google.com` or `blog.google.com.au`
- `*.google.com.*` will match `blog1.blog2.google.com.au.us` and will not match `blog1.google.com`

Exact pages on a website can be matched only if decryption is enabled for this connection:
- `xyz.com/*` will match `xyz.com/word1` and `xyz.com/word2`
- `xyz.com/word.` will only match `xyz.com/word`

As rule of thumb, in a typical scenario when you need to make sure that you allow all
possible pages, subdomains, sub-subdomains and hosts of a SaaS application with main entry point being, for example, 
`very-important-saas-app.com` you would need to create two EDL entries:
- `very-important-saas-app.com/*`
- `*.very-important-saas-app.com/*`
- `very-important-saas-app.com/`
- `*.very-important-saas-app.com/`

However for SSL/TLS exceptions scenarios, the most comprehensive match for a domain and all its hosts, subdomains and pages would only require two entries:
- `very-important-saas-app.com/`
- `*.very-important-saas-app.com/`
because subfolders and pages as such (the path after the forward slash `/`) are not present in the SNI field of the TLS/SSL handshake
