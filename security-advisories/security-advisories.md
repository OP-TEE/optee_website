---
layout: default
title: Security Advisories.
permalink: /security-advisories/
description: |-
    At this page we will list of all known security vulnerabilities found on OP-TEE.
    Likewise you will find when it was fixed and who reported the issue.

    If you have found a security issue in OP-TEE, please send us an email (see
    About) and then someone from the team will contact you for further discussion.
    The initial email doesn't have to contain any details.
---
At this page we will list of all known security vulnerabilities found on OP-TEE.
Likewise you will find when it was fixed and who reported the issue.

If you have found a security issue in OP-TEE, please send us an email (see
About) and then someone from the team will contact you for further discussion.
The initial email doesn't have to contain any details.

# December 2016
## RSA key leakage in modular exponentiation
#### Description
[Applus+ Laboratories] found out that OP-TEE is vulnerable to a timing attack
when doing the [Montgomery operations](http://www.phedny.net/papers/Timing%20attacks%20on%20RSA.pdf).

One way to optimize modular exponentiation is to make use of something called
Montgomery multiplication and Montgomery reduction. OP-TEE implements the
Montgomery operations in the big number library, libmpa. The current
implementation uses a binary Left to Right (LtoR) implementation. The LtoR
implementation is vulnerable to timing attacks since it leaks information about
the exponent in use, because it uses different amount of time in each loop when
doing the exponentiation. The leaked information can be used to completely
recover the private key. One mitigation to this attack is to change the
implementation to a constant time exponentiation algorithm instead of LtoR. One
such algorithm is the so called [Montgomery powering
ladder](https://cr.yp.to/bib/2003/joye-ladder.pdf), which does the same amount
of operations in every loop. I.e., it will always do square and multiply in
every loop. The fix (Montgomery ladder) for the timing attack has been
implemented in:  

**optee_os.git:**
- [libmpa: Implement Montgomery ladder
(40b1b281a6)](https://github.com/OP-TEE/optee_os/commit/40b1b281a6f85f8658be749dc92b57d6a8bd5e78)

| Reported by  | CVE ID | OP-TEE ID | Affected versions |
| ------------ |:------:| :-------: | ----------------- |
| [Applus+ Laboratories] | N/A | OP-TEE-2016-0003 | All versions prior to OP-TEE 2.5.0 |


## Bellcore attack
#### Description
[Applus+ Laboratories] found out that OP-TEE is vulnerable to the [Bellcore
attack](https://eprint.iacr.org/2012/553.pdf) when using fault injection /
glitching attack.

A common way to speed up RSA calculations is to use something that is called
[Chinese Remainder Theorem (CRT)](https://en.wikipedia.org/wiki/Chinese_remainder_theorem).
This optimization is also used in LibTomCrypt which is currently the default
software crypto library in OP-TEE. In short, when using CRT you are operating on
the individual prime factors 'p' and 'q' separately and then later combine them
to final result instead of just doing the exponentiation directly. However, this
also means that if somethings goes wrong in the intermediate calculations with
'p' or 'q' it is possible to completely recover the private key if you also have
access to a valid signature. I.e. it's the combination of valid and invalid
signature that makes it possible to recover the private key.  

The important thing is to never ever return any incorrect signature back to
the caller. LibTomCrypt already has mitigations for this. They have the flag
`LTC_RSA_CRT_HARDENING` which enables code that checks that the signature indeed
is valid before returning it to the user. Then there is also the flag
`LTC_RSA_BLINDING` which mixes in another random prime number when doing the
intermediate calculations. OP-TEE hasn't had those flags enabled by default in
the past and when enabling them there was some code missing related to random
number generation for big number (mpanum). The fixes for this issue can be found
in:

**optee_os.git:**
 - [ltc: Implement mp_rand for mpa_desc
(13c9b83130)](https://github.com/OP-TEE/optee_os/commit/13c9b83130e08ddd53fb3a456a678c7e3040deb9)
 - [ltc: Enable RSA_CRT_HARDENING and RSA_CRT_BLINDING
(93b0a7015c)](https://github.com/OP-TEE/optee_os/commit/93b0a7015c46d68f2bc8d1bc6c57bb6532269777).

The fix can be found in OP-TEE starting from v2.5.0.

| Reported by  | CVE ID | OP-TEE ID | Affected versions |
| ------------ |:------:| :-------: | ----------------- |
| [Applus+ Laboratories] | N/A | OP-TEE-2016-0002 | All versions prior to OP-TEE 2.5.0 |


# June 2016
## Bleichenbacher signature forgery attack
#### Description
A vulnerability in the [OP-TEE] project was found by Intel Security Advanced
Threat Research in June 2016. It appeared that OP-TEE was vulnerable to
[Bleichenbacher signature forgery attack](https://www.ietf.org/mail-archive/web/openpgp/current/msg00999.html).

The problem lies in the [LibTomCrypt] code in OP-TEE, that neglects to check
that the message length is equal to the ASN.1 encoded data length. Upstream
LibTomCrypt already had a
[fix](https://github.com/libtom/libtomcrypt/commit/5eb9743410ce4657e9d54fef26a2ee31a1b5dd0)
and there was also a [test
case](https://github.com/libtom/libtomcrypt/commit/d51715db728d99954219cc42b013db6e48db65),
verifying that the fix resolved the issue.

The fixes from upstream LibTomCrypt has been cherry-picked into OP-TEE. The fix
for TEE core can be found upstream in
[this](https://github.com/OP-TEE/optee_os/commit/30d13250c390c4f56adefdcd3b64b7cc672f9fe2)
patch and a test case has been added to the test suite for OP-TEE and that
can also be found upstream in
[this](https://github.com/OP-TEE/optee_test/commit/b58916e35fe1f73cb7d32eb5ac04ab66f59669)
patch.

| Reported by  | CVE ID | OP-TEE ID | Affected versions |
| ------------ |:------:| :-------: | ----------------- |
| [Intel Security Advanced Threat Research] | [CVE-2016-6129](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-6129) | OP-TEE-2016-0001 | All versions prior to OP-TEE v2.2.0 (fixed in OP-TEE v2.2.0) |

[Applus+ Laboratories]: http://www.appluslaboratories.com
[Intel Security Advanced Threat Research]: http://www.intelsecurity.com/advanced-threat-research
[LibTomCrypt]: http://www.libtom.org/LibTomCrypt
[optee_os]: https://github.com/OP-TEE/optee_os
[optee_test]: https://github.com/OP-TEE/optee_test
[OP-TEE]: https://github.com/OP-TEE
