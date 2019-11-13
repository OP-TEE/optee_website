---
title: Open Portable Trusted Execution Environment
permalink: "/"
description: >
    OP-TEE is an open source project which contains a full implementation to make a
    complete Trusted Execution Environment. The project has roots in a proprietary
    solution, initially created by ST-Ericsson and then owned and maintained by
    STMicroelectronics. In 2014, Linaro started working with STMicroelectronics to
    transform the proprietary TEE solution into an open source TEE solution instead.
    In September 2015, the ownership was transferred to Linaro. Today it is one of
    the key security projects in Linaro, with several of Linaro’s members supporting
    and using it.
layout: flow
jumbotron:
    triangle-divider: true
    background-image: /assets/images/background-image.jpg
    description: >-
        Isolation ● Small footprint ● Portability
    buttons:
        - title: Get Involved
          url: /contact/
          class: btn btn-primary btn-lg bottom-border-five
flow:
    - row: container_row
      sections:
          - format: text
            text_content:
                text: >
                    `OP-TEE` was initially developed as `TEE` by
                    [ST-Ericsson](http://www.stericsson.com/). In 2013, ST-Ericsson
                    achieved compliance with
                    [GlobalPlatform](https://globalplatform.org/)’s qualification, proving
                    the APIs were behaving as expected. After the [split in
                    2013](https://www.ericsson.com/en/press-releases/2013/8/ericsson-and-stmicroelectronics-complete-transaction-to-split-up-st-ericsson)
                    of [Ericsson](https://www.ericsson.com/en) and
                    [STMicroelectronics](https://www.st.com/content/st_com/en.html), TEE
                    became owned by STMicroelectronics.
          - format: text
            text_content:
                text: >
                    Later in 2013, [Linaro](https://www.linaro.org/) formed the Security Working
                    Group (SWG): one of the initial key tasks for the SWG was to work on an
                    open-source TEE project. After talking to various TEE vendors, Linaro began
                    working with the STMicroelectronics TEE project. Before we were able to
                    open-source this code, we needed to replace some proprietary components with
                    open source components. This work took several months with engineers from both
                    STMicroelectronics and Linaro's SWG making significant contributions. These
                    components included the cryptographic library, secure monitor, build system and
                    others. At the same time, the code base was cleaned up with the application of
                    [Coding
                    standards](https://optee.readthedocs.io/en/latest/general/coding_standards.html#coding-standards),
                    by running
                    [checkpatch](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/scripts/checkpatch.pl),
                    and via other related processes.
          - format: text
            text_content:
                text: >
                    On the 12th of June 2014, OP-TEE was born as an open-source project.
                    On that day, the first commit to OP-TEE's [GitHub
                    organisation](https://github.com/OP-TEE/) was pushed. Shortly
                    afterwards, a [press
                    release](https://www.linaro.org/blog/op-tee-open-source-security-mass-market/)
                    announcing the project was made. During the first year as an open
                    source project, it was owned by STMicroelectronics but jointly
                    maintained by Linaro and STMicroelectronics. In 2015, ownership of the
                    OP-TEE project was transferred from STMicroelectronics to Linaro:
                    since then, Linaro has been the primary owner and maintainer of the
                    project. Maintenance is jointly shared between Linaro, Linaro members
                    and other companies who are using OP-TEE.
          - format: title
            style: text-center
            title_content:
                size: h2
                text: Supporting Companies
          - format: members
            style: zoom grayscale
            members_content:
                item_width: "12" # bootstrap col-sm- value e.g 3, 4, 5ths etc
                items:
                    - name: Riscure
                      image:
                          path: /assets/images/riscure-logo.png
                          alt: Riscure Logo
                      url: https://www.riscure.com/
---
