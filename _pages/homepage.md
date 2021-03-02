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
  class: text-center
  title: Open Portable Trusted Execution Environment
  image: /assets/images/background-image.jpg
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
            OP-TEE is an open source Trusted Execution Enviroment (TEE) implementing the
            [Arm TrustZone technology](https://developer.arm.com/ip-products/security-ip/trustzone).
            OP-TEE has been ported to many Arm [devices and
            platforms](https://optee.readthedocs.io/en/latest/general/platforms.html).
            Originally it was developed as a proprietary TEE solution by ST-Ericsson that
            later on was moved over to STMicroelectronics.
      - format: text
        text_content:
          text: >
            Back in 2013, [Linaro](https://www.linaro.org/) formed the Security Working
            Group (SWG): one of the initial key tasks for the SWG was to work on an
            open-source TEE project. After talking to various TEE vendors, Linaro began
            working with STMicroelectronics to convert their TEE solution from being a
            proprietary TEE to become an open source TEE. After a few months of
            refactoring and rewriting major parts of the code so it was compatible with
            the BSD 2-Clause license it was released to the general public around the
            summer in 2014.
      - format: text
        text_content:
          text: >
            In 2015, ownership of the OP-TEE project was transferred from
            STMicroelectronics to Linaro. Between 2015 and 2019, Linaro was the owner of
            the project and the maintenance was shared together with the members of
            Linaro. Late 2019, Linaro transferred the OP-TEE project to the
            [TrustedFirmware.org](https://www.trustedfirmware.org/) project, who has been
            the governing body since then. Linaro is still in charge of scheduling
            releases, acting on security incidents etc. But when it comes to development
            and features there are many companies contributing to the project.
      - format: title
        style: text-center
        title_content:
          size: h2
          text: Supporting Companies
      - format: members
        style: zoom grayscale
        members_content:
          item_width: "8" # bootstrap col-sm- value e.g 3, 4, 5ths etc
          items:
            - name: Riscure
              image:
                path: /assets/images/riscure-logo.png
                alt: Riscure Logo
              url: https://www.riscure.com/
---
