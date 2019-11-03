---
layout: post
title:  "Partial Restore And Recovery"
date:   2016-07-28
categories: oracle
---

Few weeks ago I have received a request to restore five tables from a few months old backup. The client made some data modification and after the issue was noticed they wanted to restore/recover some tables.

The challenge was to perform recovery of five tables from 4TB RAC (2-node) database. With such big database problems/questions can occur, such as: do we need separate machine (to build the RAC), do we need 4TB of storage (?) to restore whole database, management approval for resources, etc.. We spent couple of days sending mails back and forth.

We came up with a solution to perform partial restore and recover on an (temporary) auxiliary instance on one of their existing systems. Since the plan was partial restore to be performed, much less space was required!

With partial restore we need to restore only system critical tablespaces such as SYSTEM, UNDO, SYSAUX and the user data tablespaces where those five tables were stored. The user data tablespaces were previously identified by the development team.

In this post I will demonstrate (step by step) how I performed partial restore and recover from backup. As a side note, we’re not using recovery catalog and CommVault is used as a backup software.

The first thing we need to identify is the backup pieces from the backup we want to restore.
Since CommVault is used for backups, having only CommVault’s job_id I’ve identified the list of backup pieces using Commvault’s getbackupList command.

The command used was: ./getbackupList -jobid job_id -outfile out_file -dbjob
In the out_file the RMAN backup pieces are reported which are part of the backup job specified for job_id.

{% highlight bash %}
DATA ----------------------------------- Archive Files List For Db Job ---------------------------------------------------------------------
archFileName                  |    fileType  |    createTime     |    archFileId|    archGroupId |    jobId
DATA ----------------------------------- Archive Files List For Db Job ---------------------------------------------------------------------
3506776_#####_q2qsnhdj_1_1    |    1         |    1454207305     |    9635006   |    121         |    3506776
3506776_#####_q3qso1q9_1_1    |    1         |    1454224070     |    9635597   |    121         |    3506776
3506776_#####_q4qsob41_1_1    |    1         |    1454233596     |    9635732   |    121         |    3506776
3506776_#####_q5qsoki8_1_1    |    1         |    1454243265     |    9635874   |    121         |    3506776
3506776_#####_q6qsot31_1_1    |    1         |    1454251995     |    9635967   |    121         |    3506776
3506776_#####_q9qsp5v1_1_1    |    1         |    1454261077     |    9636510   |    121         |    3506776
3506776_#####_qcqspkgt_1_1    |    1         |    1454276007     |    9637863   |    121         |    3506776
3506776_#####_qeqsq68e_1_1    |    1         |    1454294145     |    9639058   |    121         |    3506776
3506776_#####_qfqsq68p_1_1    |    1         |    1454294153     |    9639059   |    121         |    3506776
c-1840232663-20160201-00      |    1         |    1454294163     |    9639062   |    121         |    3506776

LOGS ----------------------------------- Archive Files List For Db Job ---------------------------------------------------------------------
archFileName                  |    fileType  |    createTime     |    archFileId|    archGroupId |    jobId
LOGS ----------------------------------- Archive Files List For Db Job ---------------------------------------------------------------------
3506776_#####_qhqsq6cm_1_1    |    4         |    1454294304     |    9639066   |    121         |    3506776
3506776_#####_qiqsq7gq_1_1    |    4         |    1454295435     |    9639129   |    121         |    3506776
3506776_#####_qjqsq88f_1_1    |    4         |    1454296191     |    9639173   |    121         |    3506776
3506776_#####_qkqsq899_1_1    |    4         |    1454296218     |    9639175   |    121         |    3506776
3506776_#####_qlqsq89h_1_1    |    4         |    1454296225     |    9639178   |    121         |    3506776
c-1840232663-20160201-01      |    4         |    1454296230     |    9639180   |    121         |    3506776
{% endhighlight %}

For the auxiliary instance, I’ve created a PFILE from the existing (production) SPFILE.
I made some modifications to be able to start the auxiliary instance on a different system.

{% highlight sql %}
SQL> startup nomount pfile='/LOCATION/initAUX.ora';

ORACLE instance started.

Total System Global Area 2137886720 bytes
Fixed Size 2254952 bytes
Variable Size 520095640 bytes
Database Buffers 1610612736 bytes
Redo Buffers 4923392 bytes
{% endhighlight %}

Next was to restore the controlfile. In this case I’ve used the backup piece created by controlfile autobackup.
The controlfile autobackup format was set to (substitution variable) %F which translates to c-‘IIIIIIIIII-YYYYMMDD-QQ’.

{% highlight sql %}
run {
allocate channel ch1 type 'sbt_tape' PARMS="SBT_LIBRARY=/appl/commvault/simpana/Base/libobk.so,BLKSIZE=262144,ENV=(CV_mmsApiVsn=2,CV_channelPar=ch1)"; restore controlfile from 'c-1840232663-20160201-01';
}
{% endhighlight %}

Once we have the controlfile we need to perform partial restore, manually specifying the database files we want to restore.
Required tablespace datafiles can be previously identified from RMAN report schema.

{% highlight sql %}
run {
allocate channel ch1 type 'sbt_tape' PARMS="SBT_LIBRARY=/appl/commvault/simpana/Base/libobk.so,BLKSIZE=262144,ENV=(CV_mmsApiVsn=2,CV_channelPar=ch1)";
set newname for datafile 1   to '/LOCATION/oradata/system.275.751110099';
set newname for datafile 2   to '/LOCATION/oradata/sysaux.316.751063097';
set newname for datafile 3   to '/LOCATION/oradata/undotbs1.314.751061299';
set newname for datafile 4   to '/LOCATION/oradata/undotbs2.277.751109205';
set newname for datafile 8   to '/LOCATION/oradata/undotbs1.318.751062777';
set newname for datafile 9   to '/LOCATION/oradata/undotbs1.279.751107163';
set newname for datafile 10  to '/LOCATION/oradata/undotbs2.276.751110001';
set newname for datafile 11  to '/LOCATION/oradata/undotbs2.278.751108533';
set newname for datafile 53  to '/LOCATION/oradata/apptbs.319.751059315';
set newname for datafile 54  to '/LOCATION/oradata/apptbs.280.751104765';
set newname for datafile 55  to '/LOCATION/oradata/apptbs.313.751060053';
set newname for datafile 82  to '/LOCATION/oradata/undotbs2.353.769707613';
set newname for datafile 191 to '/LOCATION/oradata/undotbs1.464.832685199';
restore datafile 1,2,3,4,8,9,10,11,53,54,55,82,191 from tag 'TAG20160131T032742';
}
{% endhighlight %}

Once I had the datafiles in place, I cataloged them and updated the controlfile with switch datafile … to copy command.

{% highlight sql %}
catalog datafilecopy '/LOCATION/oradata/cfpapp14-5.280.751104765';
catalog datafilecopy '/LOCATION/oradata/cfpapp14-5.313.751060053';
catalog datafilecopy '/LOCATION/oradata/cfpapp14-5.319.751059315';
catalog datafilecopy '/LOCATION/oradata/sysaux.316.751063097';
catalog datafilecopy '/LOCATION/oradata/system.275.751110099';
catalog datafilecopy '/LOCATION/oradata/undotbs1.279.751107163';
catalog datafilecopy '/LOCATION/oradata/undotbs1.314.751061299';
catalog datafilecopy '/LOCATION/oradata/undotbs1.318.751062777';
catalog datafilecopy '/LOCATION/oradata/undotbs1.464.832685199';
catalog datafilecopy '/LOCATION/oradata/undotbs2.276.751110001';
catalog datafilecopy '/LOCATION/oradata/undotbs2.277.751109205';
catalog datafilecopy '/LOCATION/oradata/undotbs2.278.751108533';
catalog datafilecopy '/LOCATION/oradata/undotbs2.353.769707613';

RMAN> switch datafile 1,2,3,4,8,9,10,11,53,54,55,82,191 to copy;

using target database control file instead of recovery catalog
datafile 1 switched to datafile copy "/LOCATION/oradata/system.275.751110099"
datafile 2 switched to datafile copy "/LOCATION/oradata/sysaux.316.751063097"
datafile 3 switched to datafile copy "/LOCATION/oradata/undotbs1.314.751061299"
datafile 4 switched to datafile copy "/LOCATION/oradata/undotbs2.277.751109205"
datafile 8 switched to datafile copy "/LOCATION/oradata/undotbs1.318.751062777"
datafile 9 switched to datafile copy "/LOCATION/oradata/undotbs1.279.751107163"
datafile 10 switched to datafile copy "/LOCATION/oradata/undotbs2.276.751110001"
datafile 11 switched to datafile copy "/LOCATION/oradata/undotbs2.278.751108533"
datafile 53 switched to datafile copy "/LOCATION/oradata/cfpapp14-5.319.751059315"
datafile 54 switched to datafile copy "/LOCATION/oradata/cfpapp14-5.280.751104765"
datafile 55 switched to datafile copy "/LOCATION/oradata/cfpapp14-5.313.751060053"
datafile 82 switched to datafile copy "/LOCATION/oradata/undotbs2.353.769707613"
datafile 191 switched to datafile copy "/LOCATION/oradata/undotbs1.464.832685199"
{% endhighlight %}

After successfully restore of the datafiles, we need to restore the archive logs in order to perform consistent recovery.
Since this was RAC database, I had to specify the sequences for each thread [1/2].
The sequence numbers I’ve previously identified from list backup of archivelog all command.

{% highlight sql %}
run {
allocate channel ch1 type 'sbt_tape' PARMS="SBT_LIBRARY=/appl/commvault/simpana/Base/libobk.so,BLKSIZE=262144,ENV=(CV_mmsApiVsn=2,CV_channelPar=ch1)";
set archivelog destination to '/LOCATION/archivelog';
restore archivelog sequence between 236182 and 236207 thread 1;
restore archivelog sequence between 246452 and 246484 thread 2;
}
{% endhighlight %}

The tricky part in this process was to run the recover database command with SKIP FOREVER TABLESPACE.

{% highlight sql %}
run {
allocate channel c1 type disk;
set until sequence 236207 thread 1;
set until sequence 246484 thread 2;
recover database skip forever tablespace tablespace_list;
}
{% endhighlight %}

And, the output should be something like:

{% highlight sql %}
allocated channel: c1
channel c1: SID=266 device type=DISK

executing command: SET until clause

executing command: SET until clause

Starting recover at 22-JUL-16

Executing: alter database datafile 65 offline drop
Executing: alter database datafile 64 offline drop
Executing: alter database datafile 7 offline drop
Executing: alter database datafile 63 offline drop
Executing: alter database datafile 36 offline drop
Executing: alter database datafile 58 offline drop
Executing: alter database datafile 59 offline drop
Executing: alter database datafile 60 offline drop
Executing: alter database datafile 61 offline drop
Executing: alter database datafile 62 offline drop
Executing: alter database datafile 81 offline drop
Executing: alter database datafile 90 offline drop
Executing: alter database datafile 91 offline drop
Executing: alter database datafile 104 offline drop
Executing: alter database datafile 113 offline drop
Executing: alter database datafile 155 offline drop
Executing: alter database datafile 178 offline drop
Executing: alter database datafile 185 offline drop
Executing: alter database datafile 189 offline drop
Executing: alter database datafile 51 offline drop
Executing: alter database datafile 50 offline drop
Executing: alter database datafile 52 offline drop
Executing: alter database datafile 45 offline drop
Executing: alter database datafile 57 offline drop
Executing: alter database datafile 44 offline drop
Executing: alter database datafile 70 offline drop
Executing: alter database datafile 43 offline drop
Executing: alter database datafile 13 offline drop
Executing: alter database datafile 30 offline drop
Executing: alter database datafile 37 offline drop
Executing: alter database datafile 38 offline drop
Executing: alter database datafile 39 offline drop
Executing: alter database datafile 40 offline drop
Executing: alter database datafile 41 offline drop
Executing: alter database datafile 42 offline drop
Executing: alter database datafile 68 offline drop
Executing: alter database datafile 69 offline drop
Executing: alter database datafile 71 offline drop
Executing: alter database datafile 72 offline drop
Executing: alter database datafile 73 offline drop
Executing: alter database datafile 74 offline drop
Executing: alter database datafile 75 offline drop
Executing: alter database datafile 76 offline drop
Executing: alter database datafile 77 offline drop
Executing: alter database datafile 78 offline drop
Executing: alter database datafile 88 offline drop
Executing: alter database datafile 89 offline drop
Executing: alter database datafile 92 offline drop
Executing: alter database datafile 93 offline drop
Executing: alter database datafile 94 offline drop
Executing: alter database datafile 99 offline drop
Executing: alter database datafile 101 offline drop
Executing: alter database datafile 102 offline drop
Executing: alter database datafile 117 offline drop
Executing: alter database datafile 118 offline drop
Executing: alter database datafile 119 offline drop
Executing: alter database datafile 120 offline drop
Executing: alter database datafile 121 offline drop
Executing: alter database datafile 126 offline drop
Executing: alter database datafile 128 offline drop
Executing: alter database datafile 129 offline drop
Executing: alter database datafile 130 offline drop
Executing: alter database datafile 131 offline drop
Executing: alter database datafile 132 offline drop
Executing: alter database datafile 133 offline drop
Executing: alter database datafile 134 offline drop
Executing: alter database datafile 135 offline drop
Executing: alter database datafile 136 offline drop
Executing: alter database datafile 137 offline drop
Executing: alter database datafile 138 offline drop
Executing: alter database datafile 139 offline drop
Executing: alter database datafile 140 offline drop
Executing: alter database datafile 141 offline drop
Executing: alter database datafile 142 offline drop
Executing: alter database datafile 143 offline drop
Executing: alter database datafile 144 offline drop
Executing: alter database datafile 145 offline drop
Executing: alter database datafile 146 offline drop
Executing: alter database datafile 147 offline drop
Executing: alter database datafile 148 offline drop
Executing: alter database datafile 149 offline drop
Executing: alter database datafile 152 offline drop
Executing: alter database datafile 153 offline drop
Executing: alter database datafile 156 offline drop
Executing: alter database datafile 157 offline drop
Executing: alter database datafile 159 offline drop
Executing: alter database datafile 160 offline drop
Executing: alter database datafile 161 offline drop
Executing: alter database datafile 162 offline drop
Executing: alter database datafile 163 offline drop
Executing: alter database datafile 164 offline drop
Executing: alter database datafile 165 offline drop
Executing: alter database datafile 166 offline drop
Executing: alter database datafile 167 offline drop
Executing: alter database datafile 168 offline drop
Executing: alter database datafile 170 offline drop
Executing: alter database datafile 171 offline drop
Executing: alter database datafile 172 offline drop
Executing: alter database datafile 173 offline drop
Executing: alter database datafile 174 offline drop
Executing: alter database datafile 177 offline drop
Executing: alter database datafile 179 offline drop
Executing: alter database datafile 182 offline drop
Executing: alter database datafile 183 offline drop
Executing: alter database datafile 187 offline drop
Executing: alter database datafile 188 offline drop
Executing: alter database datafile 190 offline drop
Executing: alter database datafile 193 offline drop
Executing: alter database datafile 194 offline drop
Executing: alter database datafile 24 offline drop
Executing: alter database datafile 32 offline drop
Executing: alter database datafile 33 offline drop
Executing: alter database datafile 79 offline drop
Executing: alter database datafile 83 offline drop
Executing: alter database datafile 116 offline drop
Executing: alter database datafile 154 offline drop
Executing: alter database datafile 180 offline drop
Executing: alter database datafile 181 offline drop
Executing: alter database datafile 12 offline drop
Executing: alter database datafile 35 offline drop
Executing: alter database datafile 67 offline drop
Executing: alter database datafile 84 offline drop
Executing: alter database datafile 85 offline drop
Executing: alter database datafile 86 offline drop
Executing: alter database datafile 87 offline drop
Executing: alter database datafile 112 offline drop
Executing: alter database datafile 123 offline drop
Executing: alter database datafile 150 offline drop
Executing: alter database datafile 169 offline drop
Executing: alter database datafile 176 offline drop
Executing: alter database datafile 186 offline drop
Executing: alter database datafile 46 offline drop
Executing: alter database datafile 47 offline drop
Executing: alter database datafile 48 offline drop
Executing: alter database datafile 49 offline drop
Executing: alter database datafile 31 offline drop
Executing: alter database datafile 66 offline drop
Executing: alter database datafile 96 offline drop
Executing: alter database datafile 95 offline drop
Executing: alter database datafile 28 offline drop
Executing: alter database datafile 27 offline drop
Executing: alter database datafile 97 offline drop
Executing: alter database datafile 98 offline drop
Executing: alter database datafile 26 offline drop
Executing: alter database datafile 80 offline drop
Executing: alter database datafile 25 offline drop
Executing: alter database datafile 56 offline drop
Executing: alter database datafile 124 offline drop
Executing: alter database datafile 125 offline drop
Executing: alter database datafile 21 offline drop
Executing: alter database datafile 22 offline drop
Executing: alter database datafile 23 offline drop
Executing: alter database datafile 29 offline drop
Executing: alter database datafile 151 offline drop
Executing: alter database datafile 20 offline drop
Executing: alter database datafile 15 offline drop
Executing: alter database datafile 16 offline drop
Executing: alter database datafile 17 offline drop
Executing: alter database datafile 18 offline drop
Executing: alter database datafile 19 offline drop
Executing: alter database datafile 34 offline drop
Executing: alter database datafile 100 offline drop
Executing: alter database datafile 103 offline drop
Executing: alter database datafile 105 offline drop
Executing: alter database datafile 106 offline drop
Executing: alter database datafile 107 offline drop
Executing: alter database datafile 108 offline drop
Executing: alter database datafile 109 offline drop
Executing: alter database datafile 110 offline drop
Executing: alter database datafile 111 offline drop
Executing: alter database datafile 115 offline drop
Executing: alter database datafile 122 offline drop
Executing: alter database datafile 127 offline drop
Executing: alter database datafile 158 offline drop
Executing: alter database datafile 175 offline drop
Executing: alter database datafile 184 offline drop
Executing: alter database datafile 14 offline drop
Executing: alter database datafile 6 offline drop
Executing: alter database datafile 114 offline drop
Executing: alter database datafile 192 offline drop
Executing: alter database datafile 5 offline drop
starting media recovery

archived log for thread 1 with sequence 236182 is already on disk as file /LOCATION/archivelog/1_236182_751706142.dbf
archived log for thread 1 with sequence 236183 is already on disk as file /LOCATION/archivelog/1_236183_751706142.dbf
archived log for thread 1 with sequence 236184 is already on disk as file /LOCATION/archivelog/1_236184_751706142.dbf
archived log for thread 1 with sequence 236185 is already on disk as file /LOCATION/archivelog/1_236185_751706142.dbf
archived log for thread 1 with sequence 236186 is already on disk as file /LOCATION/archivelog/1_236186_751706142.dbf
archived log for thread 1 with sequence 236187 is already on disk as file /LOCATION/archivelog/1_236187_751706142.dbf
archived log for thread 1 with sequence 236188 is already on disk as file /LOCATION/archivelog/1_236188_751706142.dbf
archived log for thread 1 with sequence 236189 is already on disk as file /LOCATION/archivelog/1_236189_751706142.dbf
archived log for thread 1 with sequence 236190 is already on disk as file /LOCATION/archivelog/1_236190_751706142.dbf
archived log for thread 1 with sequence 236191 is already on disk as file /LOCATION/archivelog/1_236191_751706142.dbf
archived log for thread 1 with sequence 236192 is already on disk as file /LOCATION/archivelog/1_236192_751706142.dbf
archived log for thread 1 with sequence 236193 is already on disk as file /LOCATION/archivelog/1_236193_751706142.dbf
archived log for thread 1 with sequence 236194 is already on disk as file /LOCATION/archivelog/1_236194_751706142.dbf
archived log for thread 1 with sequence 236195 is already on disk as file /LOCATION/archivelog/1_236195_751706142.dbf
archived log for thread 1 with sequence 236196 is already on disk as file /LOCATION/archivelog/1_236196_751706142.dbf
archived log for thread 1 with sequence 236197 is already on disk as file /LOCATION/archivelog/1_236197_751706142.dbf
archived log for thread 1 with sequence 236198 is already on disk as file /LOCATION/archivelog/1_236198_751706142.dbf
archived log for thread 1 with sequence 236199 is already on disk as file /LOCATION/archivelog/1_236199_751706142.dbf
archived log for thread 1 with sequence 236200 is already on disk as file /LOCATION/archivelog/1_236200_751706142.dbf
archived log for thread 1 with sequence 236201 is already on disk as file /LOCATION/archivelog/1_236201_751706142.dbf
archived log for thread 1 with sequence 236202 is already on disk as file /LOCATION/archivelog/1_236202_751706142.dbf
archived log for thread 1 with sequence 236203 is already on disk as file /LOCATION/archivelog/1_236203_751706142.dbf
archived log for thread 1 with sequence 236204 is already on disk as file /LOCATION/archivelog/1_236204_751706142.dbf
archived log for thread 1 with sequence 236205 is already on disk as file /LOCATION/archivelog/1_236205_751706142.dbf
archived log for thread 1 with sequence 236206 is already on disk as file /LOCATION/archivelog/1_236206_751706142.dbf
archived log for thread 2 with sequence 246453 is already on disk as file /LOCATION/archivelog/2_246453_751706142.dbf
archived log for thread 2 with sequence 246454 is already on disk as file /LOCATION/archivelog/2_246454_751706142.dbf
archived log for thread 2 with sequence 246455 is already on disk as file /LOCATION/archivelog/2_246455_751706142.dbf
archived log for thread 2 with sequence 246456 is already on disk as file /LOCATION/archivelog/2_246456_751706142.dbf
archived log for thread 2 with sequence 246457 is already on disk as file /LOCATION/archivelog/2_246457_751706142.dbf
archived log for thread 2 with sequence 246458 is already on disk as file /LOCATION/archivelog/2_246458_751706142.dbf
archived log for thread 2 with sequence 246459 is already on disk as file /LOCATION/archivelog/2_246459_751706142.dbf
archived log for thread 2 with sequence 246460 is already on disk as file /LOCATION/archivelog/2_246460_751706142.dbf
archived log for thread 2 with sequence 246461 is already on disk as file /LOCATION/archivelog/2_246461_751706142.dbf
archived log for thread 2 with sequence 246462 is already on disk as file /LOCATION/archivelog/2_246462_751706142.dbf
archived log for thread 2 with sequence 246463 is already on disk as file /LOCATION/archivelog/2_246463_751706142.dbf
archived log for thread 2 with sequence 246464 is already on disk as file /LOCATION/archivelog/2_246464_751706142.dbf
archived log for thread 2 with sequence 246465 is already on disk as file /LOCATION/archivelog/2_246465_751706142.dbf
archived log for thread 2 with sequence 246466 is already on disk as file /LOCATION/archivelog/2_246466_751706142.dbf
archived log for thread 2 with sequence 246467 is already on disk as file /LOCATION/archivelog/2_246467_751706142.dbf
archived log for thread 2 with sequence 246468 is already on disk as file /LOCATION/archivelog/2_246468_751706142.dbf
archived log for thread 2 with sequence 246469 is already on disk as file /LOCATION/archivelog/2_246469_751706142.dbf
archived log for thread 2 with sequence 246470 is already on disk as file /LOCATION/archivelog/2_246470_751706142.dbf
archived log for thread 2 with sequence 246471 is already on disk as file /LOCATION/archivelog/2_246471_751706142.dbf
archived log for thread 2 with sequence 246472 is already on disk as file /LOCATION/archivelog/2_246472_751706142.dbf
archived log for thread 2 with sequence 246473 is already on disk as file /LOCATION/archivelog/2_246473_751706142.dbf
archived log for thread 2 with sequence 246474 is already on disk as file /LOCATION/archivelog/2_246474_751706142.dbf
archived log for thread 2 with sequence 246475 is already on disk as file /LOCATION/archivelog/2_246475_751706142.dbf
archived log for thread 2 with sequence 246476 is already on disk as file /LOCATION/archivelog/2_246476_751706142.dbf
archived log for thread 2 with sequence 246477 is already on disk as file /LOCATION/archivelog/2_246477_751706142.dbf
archived log for thread 2 with sequence 246478 is already on disk as file /LOCATION/archivelog/2_246478_751706142.dbf
archived log for thread 2 with sequence 246479 is already on disk as file /LOCATION/archivelog/2_246479_751706142.dbf
archived log for thread 2 with sequence 246480 is already on disk as file /LOCATION/archivelog/2_246480_751706142.dbf
archived log for thread 2 with sequence 246481 is already on disk as file /LOCATION/archivelog/2_246481_751706142.dbf
archived log for thread 2 with sequence 246482 is already on disk as file /LOCATION/archivelog/2_246482_751706142.dbf
archived log for thread 2 with sequence 246483 is already on disk as file /LOCATION/archivelog/2_246483_751706142.dbf
archived log file name=/LOCATION/archivelog/1_236182_751706142.dbf thread=1 sequence=236182
archived log file name=/LOCATION/archivelog/2_246453_751706142.dbf thread=2 sequence=246453
archived log file name=/LOCATION/archivelog/1_236183_751706142.dbf thread=1 sequence=236183
archived log file name=/LOCATION/archivelog/2_246454_751706142.dbf thread=2 sequence=246454
archived log file name=/LOCATION/archivelog/1_236184_751706142.dbf thread=1 sequence=236184
archived log file name=/LOCATION/archivelog/1_236185_751706142.dbf thread=1 sequence=236185
archived log file name=/LOCATION/archivelog/2_246455_751706142.dbf thread=2 sequence=246455
archived log file name=/LOCATION/archivelog/2_246456_751706142.dbf thread=2 sequence=246456
archived log file name=/LOCATION/archivelog/1_236186_751706142.dbf thread=1 sequence=236186
archived log file name=/LOCATION/archivelog/2_246457_751706142.dbf thread=2 sequence=246457
archived log file name=/LOCATION/archivelog/2_246458_751706142.dbf thread=2 sequence=246458
archived log file name=/LOCATION/archivelog/2_246459_751706142.dbf thread=2 sequence=246459
archived log file name=/LOCATION/archivelog/1_236187_751706142.dbf thread=1 sequence=236187
archived log file name=/LOCATION/archivelog/2_246460_751706142.dbf thread=2 sequence=246460
archived log file name=/LOCATION/archivelog/2_246461_751706142.dbf thread=2 sequence=246461
archived log file name=/LOCATION/archivelog/1_236188_751706142.dbf thread=1 sequence=236188
archived log file name=/LOCATION/archivelog/1_236189_751706142.dbf thread=1 sequence=236189
archived log file name=/LOCATION/archivelog/2_246462_751706142.dbf thread=2 sequence=246462
archived log file name=/LOCATION/archivelog/1_236190_751706142.dbf thread=1 sequence=236190
archived log file name=/LOCATION/archivelog/1_236191_751706142.dbf thread=1 sequence=236191
archived log file name=/LOCATION/archivelog/1_236192_751706142.dbf thread=1 sequence=236192
archived log file name=/LOCATION/archivelog/2_246463_751706142.dbf thread=2 sequence=246463
archived log file name=/LOCATION/archivelog/1_236193_751706142.dbf thread=1 sequence=236193
archived log file name=/LOCATION/archivelog/2_246464_751706142.dbf thread=2 sequence=246464
archived log file name=/LOCATION/archivelog/1_236194_751706142.dbf thread=1 sequence=236194
archived log file name=/LOCATION/archivelog/2_246465_751706142.dbf thread=2 sequence=246465
archived log file name=/LOCATION/archivelog/2_246466_751706142.dbf thread=2 sequence=246466
archived log file name=/LOCATION/archivelog/2_246467_751706142.dbf thread=2 sequence=246467
archived log file name=/LOCATION/archivelog/1_236195_751706142.dbf thread=1 sequence=236195
archived log file name=/LOCATION/archivelog/1_236196_751706142.dbf thread=1 sequence=236196
archived log file name=/LOCATION/archivelog/2_246468_751706142.dbf thread=2 sequence=246468
archived log file name=/LOCATION/archivelog/2_246469_751706142.dbf thread=2 sequence=246469
archived log file name=/LOCATION/archivelog/1_236197_751706142.dbf thread=1 sequence=236197
archived log file name=/LOCATION/archivelog/1_236198_751706142.dbf thread=1 sequence=236198
archived log file name=/LOCATION/archivelog/1_236199_751706142.dbf thread=1 sequence=236199
archived log file name=/LOCATION/archivelog/2_246470_751706142.dbf thread=2 sequence=246470
archived log file name=/LOCATION/archivelog/2_246471_751706142.dbf thread=2 sequence=246471
archived log file name=/LOCATION/archivelog/2_246472_751706142.dbf thread=2 sequence=246472
archived log file name=/LOCATION/archivelog/1_236200_751706142.dbf thread=1 sequence=236200
archived log file name=/LOCATION/archivelog/2_246473_751706142.dbf thread=2 sequence=246473
archived log file name=/LOCATION/archivelog/2_246474_751706142.dbf thread=2 sequence=246474
archived log file name=/LOCATION/archivelog/1_236201_751706142.dbf thread=1 sequence=236201
archived log file name=/LOCATION/archivelog/2_246475_751706142.dbf thread=2 sequence=246475
archived log file name=/LOCATION/archivelog/1_236202_751706142.dbf thread=1 sequence=236202
archived log file name=/LOCATION/archivelog/1_236203_751706142.dbf thread=1 sequence=236203
archived log file name=/LOCATION/archivelog/2_246476_751706142.dbf thread=2 sequence=246476
archived log file name=/LOCATION/archivelog/2_246477_751706142.dbf thread=2 sequence=246477
archived log file name=/LOCATION/archivelog/2_246478_751706142.dbf thread=2 sequence=246478
archived log file name=/LOCATION/archivelog/1_236204_751706142.dbf thread=1 sequence=236204
archived log file name=/LOCATION/archivelog/1_236205_751706142.dbf thread=1 sequence=236205
archived log file name=/LOCATION/archivelog/2_246479_751706142.dbf thread=2 sequence=246479
archived log file name=/LOCATION/archivelog/2_246480_751706142.dbf thread=2 sequence=246480
archived log file name=/LOCATION/archivelog/2_246481_751706142.dbf thread=2 sequence=246481
archived log file name=/LOCATION/archivelog/1_236206_751706142.dbf thread=1 sequence=236206
archived log file name=/LOCATION/archivelog/2_246482_751706142.dbf thread=2 sequence=246482
archived log file name=/LOCATION/archivelog/2_246483_751706142.dbf thread=2 sequence=246483
media recovery complete, elapsed time: 00:20:54
Finished recover at 22-JUL-16
released channel: c1
{% endhighlight %}

After successful recover don’t forget to update the redo log files definition.
I’ve dropped some redo log groups leaving the database with minimum of (2 groups) two redo log file members per group (to save on space).

_And ..._

{% highlight sql %}
SQ> alter database open resetlogs;

Database altered.
{% endhighlight %}

Moving data from restored database to production database.

For moving the data I’ve used DB Link which was very straight forward and I’ve used CTAS to move the required 5 tables.

Partial restore is very useful if you need to restore (extract) small portion of your backups.
Also, if the backup of the database is done with FILESPERSET=1 it would take much less time, since you’ll be scanning only the backupsets from which you need to restore.