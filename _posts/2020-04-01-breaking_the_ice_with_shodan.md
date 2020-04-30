---
layout: post
title: "Breaking the ice with Shodan"
categories: misc
---

### Shodan.io
Shodan defines itself as the search engine for _Internet-connected devices_. Replace _Internet-connected devices_ with webcams, refrigerators, power plants... Raspberry Pis... you-name-it.

I'd always been curious about Shodan but not that curious to break into someone else's webcam, not my cup of tea. What changed my mind?
In the last few days I've noticed an unusual and very suspect surge of messages like the followin one on a couple of servers which I'd exposed to the Internet:
```
Mar 31 15:50:38 <server's name> sshd[11391]: Invalid user miguel from 122.51.114.213 port 48438
Mar 31 15:50:38 <server's name> sshd[11391]: pam_unix(sshd:auth): check pass; user unknown
Mar 31 15:50:38 <server's name> sshd[11391]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=122.51.114.213
Mar 31 15:50:41 <server's name> sshd[11391]: Failed password for invalid user miguel from 122.51.114.213 port 48438 ssh2
Mar 31 15:50:42 <server's name> sshd[11391]: Received disconnect from 122.51.114.213 port 48438:11: Bye Bye [preauth]
Mar 31 15:50:42 <server's name> sshd[11391]: Disconnected from invalid user miguel 122.51.114.213 port 48438 [preauth]
```

#### Who's Miguel?
Do not be fooled by the names being used for the login attempts. There are no _miguel_ or _xiaobing_ or _p@sswd@123456_ behind the attack. More realistically, someone has put together a list of most common user names with a list of most commonly used passwords and is looping through the two of them.
Worrisome? A bit.
Dangerous? Possibly.
Annoying? YES!

Apart from strongly recommending some serious hardening to your Internet-facing devices, this has prompted me to get a bit more familiar with Shodan and take a look at myself _from the other side_. A bit like those _public profile_ pages so common on social websites.
What did I do? I searched for myself on Shodan to check what kind of footprints I'd be leaving on the Internet.

To get familiar with Shodan and it's GUI I recommend reading the floowing article by [Daniel Miessler](https://danielmiessler.com/): [A Shodan Tutorial and Primer](https://danielmiessler.com/study/shodan/).
You can find all sort of things in there. Nice huh?

#### Where am I?
Well, being interested in automating my activities as much as possible, I've headed to [Shodan API](https://developer.shodan.io/) and, humbly following some of the examples, I've built a simple Python script whose purpose is to query the API and return some results.

Let's take a look at the high-level steps:
1. install the library, `easy_install shodan`
2. import the library, `import shodan`
3. run it, `api.search(<query string>)`

How easy is that?

Let's take a look at the short script, `apiquery.py`:
```
#!/usr/bin/env python3

import shodan
import sys

SHODAN_API_KEY = "<API key here>"

if len(sys.argv) == 1:
        print('Usage: %s <search query>' % sys.argv[0])
        sys.exit(1)

try:
    # Search Shodan
    api = shodan.Shodan(SHODAN_API_KEY)

    query = ' '.join(sys.argv[1:])
    results = api.search(query)


    # Show the results
    print('Results found: {}'.format(results['total']))
    for result in results['matches']:
        print('IP: {}'.format(result['ip_str']))
        print('Port: {}'.format(result['port']))
        print(result['data'])
        print('')
except shodan.APIError as e:
    print('Error: {}'.format(e))
```

Let's run the script:
```
$ ./apiquery.py country:IT city:"Milan"
Results found: 286058
```

286k results in my city alone! Let's restrict the query:
```
$ ./apiquery.py country:IT city:"Milan" port:22
Results found: 7259
```

A bit more manageable but still interesting outcome, right? If I knew what my public address would be I could restrict even more as follows:
```
$ ./apiquery.py country:IT city:"Milan" port:22 net:<ip_addr>
Results found: 1
IP: <ip_addr>
Port: 22
SSH-2.0-dropbear
Key type: ssh-rsa
Key: AAAAB3NzaC1yc...
Fingerprint: 30:...:cf

Kex Algorithms:
	curve25519-sha256@libssh.org
	ecdh-sha2-nistp521
	ecdh-sha2-nistp384
	ecdh-sha2-nistp256
	diffie-hellman-group14-sha1
	diffie-hellman-group1-sha1
	kexguess2@matt.ucc.asn.au

Server Host Key Algorithms:
	ssh-rsa

Encryption Algorithms:
	aes128-ctr
	aes256-ctr
	aes128-cbc
	aes256-cbc
	twofish256-cbc
	twofish-cbc
	twofish128-cbc
	3des-ctr
	3des-cbc

MAC Algorithms:
	hmac-sha1-96
	hmac-sha1
	hmac-sha2-256
	hmac-sha2-512
	hmac-md5

Compression Algorithms:
	none
```

Of course `port:22` can be removed to display any ports/services I'm currently exposing.
