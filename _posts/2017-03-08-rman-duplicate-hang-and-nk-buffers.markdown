---
layout: post
title:  "RMAN Duplicate Hang And nK Buffers"
date:   2017-03-08
categories: oracle
---

This has been sitting in draft for quite a long.

Few days ago one of my colleague experienced interesting hang situation during database duplicate.
After successfully datafiles restore the RMAN session went to hang state on the step when the instance tries to recover (apply archive logs) the datafiles.

The trace file contained information like: _Recovery is unrecoverable because it fails to get a free buffer._
The SPFILE from target database was copied and it turned out that the on target database nK buffer cache was defined for different size than default block size.