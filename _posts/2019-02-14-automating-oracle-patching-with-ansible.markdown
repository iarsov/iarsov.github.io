---
layout: post
title:  "Automating Oracle Patching With Ansible"
date:   2019-02-14
categories: oracle
---

Recently I wrote a blog post for an Ansible module I wrote for Oracle patching. The blog post is published on Pythian’s blog.

You can read more at <https://blog.pythian.com/automating-oracle-patching-with-an-ansible-module>{:target="_blank"}

The idea of the playbook and the module is to automate whole process when patching Oracle binaries with PSUs, BPs, RUs etc.

You can find the playbook and module on my GitHub page at <https://github.com/iarsov/ansible-orapatch>{:target="_blank"}

README is available at <https://github.com/iarsov/ansible-orapatch/blob/master/README.md>{:target="_blank"}

Expected actions performed by the module:

The module will identify what database instances, listeners and ASM instances are running.<br/>
The module will shutdown all listeners and database instances, only if the home from which services are running is being patched.<br/>
The module will start up all previously stopped services after it completes with patching *<br/>
The module will skip databases which are not in READ WRITE state **<br/>
The module will identify if a given database is in STANDBY or PRIMARY role ***<br/>
The module always patches GI homes with opatchauto<br/>
The module always patches DB homes with opatch<br/>
The module will make multiple restarts of the databases and listeners during the process<br/>

*Assuming no error occurred and module did not fail during the patching process.<br/>
** Even if the databases are specified for patching<br/>
*** Databases in STANDBY role are not patched
