---
title: Security Advisories.
permalink: "/security-advisories/"
layout: jumbotron-container
description: |-
  At this page we will list of all known security vulnerabilities found on OP-TEE.
  Likewise you will find when it was fixed and who reported the issue.

  If you have found a security issue in OP-TEE, please send us an email (see
  About) and then someone from the team will contact you for further discussion.
  The initial email doesn't have to contain any details.
jumbotron:
    triangle-divider: true
    title: Security Advisories
    description: >
        At this page we will list of all known security vulnerabilities found on OP-TEE.
        Likewise you will find when it was fixed and who reported the issue.
    animation: fade
    background-image: /assets/images/background-image.jpg
---
At this page we will list of all known security vulnerabilities found on OP-TEE.
Likewise you will find when it was fixed and who reported the issue.

If you have found a security issue in OP-TEE, please send us an email (see
About) and then someone from the team will contact you for further discussion.
The initial email doesn't have to contain any details.

# May 2018
## Spectre variant 4 (CVE-2018-3639)
#### Current status:
In the affected Arm cores (Cortex-A57, Cortex-A72, Cortex-A73 and Cortex-A75)
who all are Armv8 based there are configuration control registers
available at EL3 that when enabled effectively mitigate a potential
Spectre v4 attack. This means that the mitigation for this is not being
implemented at S-EL1 where the TEE resides. For more information about
the EL-3 mitigations, please see the [Trusted Firmware A Security Advisory TFV 7](https://github.com/ARM-software/arm-trusted-firmware/wiki/Trusted-Firmware-A-Security-Advisory-TFV-7).
In all officially supported Armv8 OP-TEE setups we are using TF-A as
the firmware and therefore we consider that the TF-A mitigations at
EL-3 effectively stop Spectre v4 attacks in a system running OP-TEE and
TF-A.

As mentioned in the
[whitepaper](https://developer.arm.com/support/arm-security-updates/speculative-processor-vulnerability)
from Arm about these types of attacks, there are new barriers (SSBB and PSSBB)
being introduced also. These could also be used as a mitigation
directly at lower exception levels. But just as for Spectre v1, this
involves manual inspection of code and placement of barriers until tooling
has become better to figure out this on its own. This manual work is
error prone and very time consuming and has to be done over and over
again. We have been doing some manual inspection of the OP-TEE code and
so far have not been able to identify and vulnerable areas. But just as
for Spectre v1, we continuously discuss tooling etc with members of Linaro.

| Reported by  | CVE ID | OP-TEE ID | Affected versions |
| ------------ |:------:| :-------: | ----------------- |
| [Google Project Zero] | [CVE-2018-3639](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-3639) | N/A | N/A (EL-3 TF-A implements the mitigation) |


# January 2018
## Meltdown and Spectre
In collaboration a group of different people (see "Reported by") have found out
that it is possible to circumvent security countermeasures and privilege
escalation by using speculative execution, caches, out-of-order execution in a
specific way. All details about the attacks has been thoroughly described in
the whitepapers that can found in the [Meltdown and Spectre] page. A
[blog post](https://www.linaro.org/blog/meltdown-spectre/) is also available on
the Linaro website. So we will not cover the details here, instead we will
highlight how it could affect OP-TEE and what the mitigations are.

### Variant 1: bounds check bypass (CVE-2017-5753)

#### Possible attack
Since user data provided to Trusted Applications most often comes from
non-secure side, it is important to check the code where we are using those
non-secure parameters. The same type of checks are necessary when doing
syscalls from Trusted Applications. In principle, this means that non-secure
side eventually could access secure memory when untrusted value is passed to
secure side.

#### Current status:
We have been doing some manual inspection of the OP-TEE code, and so far have
not been able to identify any vulnerable areas. Code analysis tools and
compiler update are being discussed with members of Linaro.

| Reported by  | CVE ID | OP-TEE ID | Affected versions |
| ------------ |:------:| :-------: | ----------------- |
| [Google Project Zero], [University of Pennsylvania], [University of Maryland], [Rambus], [Graz University of Technology], [University of Adelaide], [Data61] | [CVE-2017-5753](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-5753) | OP-TEE-2018-0001 | All versions |

### Variant 2: branch target injection (CVE-2017-5715)

#### Possible attack
In theory it would be possible for a program in non-secure world to train the
branch predictor to trick the secure monitor to speculatively read secure
memory and as a consequence of that leak information to the cache that can be
observed by a less privileged process. To exploit this an attacker needs to
find a gadget that can be used as a trampoline to get access kernel memory
(from a Trusted Application for example).

The mitigation here is to invalidate the branch predictor when:
* Going from non-secure to the secure environment.
* When doing syscall from S-EL0 to S-EL1.

#### Current status:
For `Armv8-A` builds we are typically running OP-TEE with Arm Trusted Firmware,
patches can be found here:
* https://github.com/ARM-software/arm-trusted-firmware/pull/1214 (merged)

For builds where we are not using Arm TF (typically `Armv7-A` builds) we have
implemented mitigations that can be found here:
* https://github.com/OP-TEE/optee_os/pull/2047 (merged)
* https://github.com/OP-TEE/optee_os/pull/2065 (merged)

For SVC calls, we have patches here:
* https://github.com/OP-TEE/optee_os/pull/2055 (`Armv7-A`, `AArch32`) (merged)
* https://github.com/OP-TEE/optee_os/pull/2072 (`AArch64`) (merged) and
https://github.com/OP-TEE/optee_os/pull/2229 (`AArch64`) (merged)

| Reported by  | CVE ID | OP-TEE ID | Affected versions |
| ------------ |:------:| :-------: | ----------------- |
| [Google Project Zero], [University of Pennsylvania], [University of Maryland], [Rambus], [Graz University of Technology], [University of Adelaide], [Data61] | [CVE-2017-5715](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-5715) | OP-TEE-2018-0002 | All versions prior to OP-TEE 3.0.0 (32 bits) or 3.1.0 (64 bits) |

### Variant 3: rogue data cache load (CVE-2017-5754)

#### Possible attack
Just as in Linux kernel it could be possible to do the same type of attack from
a Trusted Application as being described in the [Meltdown whitepaper]. I.e.,
under some conditions the CPU would read and execute instructions speculatively
before the CPU handles the illegal access (traps).

#### Current status:
Our patches can be found here:
* https://github.com/OP-TEE/optee_os/pull/2048 (merged)

The mitigation ideas are the same as with [KPTI], i.e, we keep the amount of
kernel memory being mapped to a minimum when running in usermode. It should
also be noted that there are currently no known devices running OP-TEE who are
susceptible to the Meltdown attack. Still we have decided to move on and merged
the mitigation patches, since we believe that this gives additional security and
it also means that we are prepared if/when we find OP-TEE running on
Cortex-A75.

| Reported by  | CVE ID | OP-TEE ID | Affected versions |
| ------------ |:------:| :-------: | ----------------- |
| [Google Project Zero], [Cyberus Technology], [Graz University of Technology] | [CVE-2017-5754](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-5754) | OP-TEE-2018-0003 | All versions prior to OP-TEE 3.0.0 |

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
| [Applus+ Laboratories] | [CVE-2017-1000413](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-1000413) | OP-TEE-2016-0003 | All versions prior to OP-TEE 2.5.0 |


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
| [Applus+ Laboratories] | [CVE-2017-1000412](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-1000412) | OP-TEE-2017-0002 | All versions prior to OP-TEE 2.5.0 |


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
[Cyberus Technology]: https://www.cyberus-technology.de
[Data61]: https://www.data61.csiro.au
[Google Project Zero]: https://googleprojectzero.blogspot.com
[Graz University of Technology]: https://www.iaik.tugraz.at
[Intel Security Advanced Threat Research]: http://www.intelsecurity.com/advanced-threat-research
[KPTI]: https://lwn.net/Articles/741878
[LibTomCrypt]: http://www.libtom.org/LibTomCrypt
[Meltdown and Spectre]: https://spectreattack.com
[Meltdown whitepaper]: https://meltdownattack.com/meltdown.pdf
[optee_os]: https://github.com/OP-TEE/optee_os
[optee_test]: https://github.com/OP-TEE/optee_test
[OP-TEE]: https://github.com/OP-TEE
[Rambus]: https://www.rambus.com
[University of Adelaide]: https://www.adelaide.edu.au
[University of Maryland]: https://www.umd.edu
[University of Pennsylvania]: https://www.upenn.edu

