---
layout: post
title:  "Troubleshooting TNS-12531 (TNS:Cannot Allocate Memory Error)"
date:   2015-01-14
categories: oracle
---

With this post I would like to share some troubleshooting information regarding TNS-12531. Today I received call to check one of the servers because the database was not accessible, when I logged on and checked the space available on the file system I was surprised to see the LVM mount point intended for oracle to be 100% full. When I checked what was causing the disk to fill it turned out to be the listener xml log files. Oracle was generating TNS-12531: TNS:cannot allocate memory error messages as crazy.

The tnslsnr process was stuck and those few times that I tried to start the listener I was waiting forever on
Starting /u01/app/grid/product/12.1.0/grid/bin/tnslsnr: please wait…

If you see the definition for TNS-12531 it says it’s due to insufficient memory/resources and most of the solutions are related to resource/memory configuration. But, that was not the problem this time because the server had enough memory. I’ve configured tracing for the listener in order to see in more details what was causing the error.

I’ve set TRACE_LEVEL_listener_name and tried to start the listener in order to populate the trace file.
This is some of the output from listener.trc

{% highlight bash %}
...
        INCOMING CALL
[14-JAN-2015 16:09:46:597] nsevrec: event is 0x1, on 1
[14-JAN-2015 16:09:46:597] nsevwait: 1 posted event(s)
[14-JAN-2015 16:09:46:597] nsevwait: exit (0)
[14-JAN-2015 16:09:46:597] nsglhe: entry
[14-JAN-2015 16:09:46:597] nsglhe: Event on cxd 0x11f72c0.
[14-JAN-2015 16:09:46:598] nsglhc: Allocating cxd 0x1212810
[14-JAN-2015 16:09:46:598] nsanswer: entry
[14-JAN-2015 16:09:46:598] nsopen: entry
[14-JAN-2015 16:09:46:598] nsmal: entry
[14-JAN-2015 16:09:46:598] nsmal: 1576 bytes at 0x1225cb0
[14-JAN-2015 16:09:46:598] nsmal: normal exit
[14-JAN-2015 16:09:46:598] nsopenmplx: entry
[14-JAN-2015 16:09:46:598] snlinGetAddrInfo: entry
[14-JAN-2015 16:09:46:598] snlinGetAddrInfo: getaddrinfo() failed with error -2      
[14-JAN-2015 16:09:46:598] snlinGetAddrInfo: exit
[14-JAN-2015 16:09:46:598] nserror: entry
...
{% endhighlight %}

As you can there was an error for gettaddrinfo() with error number -2, getaddrinfo() is used for network address and service translation and if it succeeds will return 0, otherwise will return error code number. Error codes for getaddrinfo() can be found in netdb.h stored in /usr/include where header files for C compiler are stored. I’ve ran grep to find the error code which is EAI_NONAME.

{% highlight bash %}
cat /usr/include/netdb.h | grep 'EAI_'

# define EAI_BADFLAGS     -1    /* Invalid value for `ai_flags' field.  */
# define EAI_NONAME       -2    /* NAME or SERVICE is unknown.  */
# define EAI_AGAIN        -3    /* Temporary failure in name resolution.  */
# define EAI_FAIL         -4    /* Non-recoverable failure in name res.  */
# define EAI_FAMILY       -6    /* `ai_family' not supported.  */
# define EAI_SOCKTYPE     -7    /* `ai_socktype' not supported.  */
# define EAI_SERVICE      -8    /* SERVICE not supported for `ai_socktype'.  */
# define EAI_MEMORY       -10   /* Memory allocation failure.  */
# define EAI_SYSTEM       -11   /* System error returned in `errno'.  */
# define EAI_OVERFLOW     -12   /* Argument buffer overflow.  */
#  define EAI_NODATA      -5    /* No address associated with NAME.  */
#  define EAI_ADDRFAMILY  -9    /* Address family for NAME not supported.  */
#  define EAI_INPROGRESS  -100  /* Processing request in progress.  */
#  define EAI_CANCELED    -101  /* Request canceled.  */
#  define EAI_NOTCANCELED -102  /* Request not canceled.  */
#  define EAI_ALLDONE     -103  /* All requests done.  */
#  define EAI_INTR        -104  /* Interrupted by a signal.  */
#  define EAI_IDN_ENCODE  -105  /* IDN encoding failed.  */
{% endhighlight %}

The [description](http://linux.die.net/man/3/getaddrinfo){:target="_blank"} for EAI_NONAME is: The node or service is not known; or both node and service are NULL; or AI_NUMERICSERV was specified in hints.ai_flags and service was not a numeric port-number string.

By node it means hostname, when I checked the definition for hostname I found the problem, which in this case it turned out to be hostname definition which was different from the ip-hostname definition in /etc/hosts. It was clear where the problem was, after fixing the hostname definition the listener started successfully.