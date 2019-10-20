---
layout: page
title: Orapatch
permalink: /orapatch/
---

Ansible custom module and playbook for automated Oracle patching.

[Go to GitHub repository](https://github.com/iarsov/ansible-orapatch){:target="_blank"}

Readme snippet:

The main purpose of the module is to automate the patching process of Oracle database and grid infrastructure binaries with PSUs, BPs and RUs released by Oracle.

One-off patches: It won’t work with one-off patches as it’s not designed for that. Though, it that can be extended to support one-off patches.

The module will use opatchauto if the Oracle home being patched is grid infrastructure, otherwise it will use standard opatch steps.

The patching is customizable via role’s variables definition. For example, you can run just prerequisites without applying the patch, patch binaries without database dictionary changes, skip the OJVM patch etc.

The module supports 11g, 12c and 18c database versions. It should work properly on 10g as well, but I haven’t tested it.

Expected actions performed by the module:

The module will identify what database instances, listeners and ASM instances are running.
The module will shutdown all listeners and database instances, only if the home from which services are running is being patched.
The module will start up all previously stopped services after it completes with patching*
The module will skip databases which are not in READ WRITE state**
The module will identify if a given database is in STANDBY or PRIMARY role***
The module always patches GI homes with opatchauto
The module always patches DB homes with opatch
The module will make multiple restarts of the databases and listeners during the process

    Assuming no error occurred and module did not fail during the patching process.
    ** Even if the databases are specified for patching
    *** Databases in STANDBY role are not patched

Note: If an error is encountered and you restart the process, the module will not automatically start previously stopped services. The module will note stopped services at the beginning of the process and it will leave the services stopped at the end of execution. Due to the nature of how Oracle patching is performed, in some cases if something breaks a manual intervention might be needed. In other words if you restart the Ansible process do not expect to continue from where it stopped.