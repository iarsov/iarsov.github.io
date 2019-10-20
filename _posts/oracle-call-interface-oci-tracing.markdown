---
layout: post
title:  "Oracle Call Interface (OCI) Tracing"
date:   2017-04-10
categories: oracle
---

The Oracle Call Interface (OCI) is a set of APIs which provides interaction with an Oracle database. It supports all phases of a SQL statement execution.

If you ever wondered how to trace OCI function calls you can do it by setting EVENT_10842 environment variable.

{% highlight shell %}
EVENT_10842="server=<>;user=<>;stmt=<>;level=<>"
{% endhighlight %}

Further you can have the trace files generated in a specific directory by setting ORA_CLIENTTRACE_DIR=directory_path.
For ORA_CLIENTTRACE_DIR to take effect Automatic Diagnostic Repository (ADR) has to be disabled. You can disable it by setting DIAG_ADR_ENABLED=off in your sqlnet.ora.

A tracefile is generated for each connection in the format of ora_skgu_<pid>.trc1 where pid is the process id of the connection on the (client) system.

I will demonstrate an example trace of a DML statement, SELECT statement and one statement which throws an error. The OCI function calls varies, depending on the statement that’s executed. For example if we have SELECT statement we should see OCI fetch calls.

{% highlight shell %}
# 04/09/17 23:27:59:615 # Thread ID 130486272 # Entry - OCIHandleAlloc(parenth = 0x7fde708b6e00, hndlp = 0x7fde708c2160, htype = service hndl, xtramem_sz =0, usermemp = 0x0, flag = TRUE);
# 04/09/17 23:27:59:615 # Thread ID 130486272 # Exit - OCIHandleAlloc
# 04/09/17 23:27:59:695 # Thread ID 130486272 # Exit - OCIAttrSet
# 04/09/17 23:27:59:695 # Thread ID 130486272 # Entry - OCIAttrSet(trgthndlp = 0x7fde7119f380, trghndltyp = 9, attributep = 0x0, size = 5, errhp = 0x7fde702f5a10);
# 04/09/17 23:27:59:696 # Thread ID 130486272 # Exit - OCIAttrSet
# 04/09/17 23:27:59:696 # Thread ID 130486272 # Entry - OCIAttrSet(trgthndlp = 0x7fde7119f380, trghndltyp = 9, attributep = 0x7fde6f553f70, size = 7, errhp = 0x7fde702f5a10);
# 04/09/17 23:27:59:696 # Thread ID 130486272 # Exit - OCIAttrSet
# 04/09/17 23:27:59:697 # Thread ID 130486272 # Entry - OCISessionBegin(svchp = 0x7fde702f5928, errhp = 0x7fde702f5a10, usrhp = 0x7fde7119f380,  : user = ARSOV, scredt = OCI_CRED_RDBMS, mode = OCI_DEFAULT(000000000));
# 04/09/17 23:27:59:723 # Thread ID 130486272 # Exit - OCISessionBegin
{% endhighlight %}

At the beginning of the trace file we have the OCIHandleAlloc – function which returns the pointer to the allocated handle.
The type of the handle is htype = service hndl which stands for Service Handle. Under the service context handle there are additional three handles allocated:

    Server handle: establish the physical database connection
    User session handle: defines the user’s domain (privileges, roles etc.)
    Transaction handle: context in which statements are executed, PL/SQL packages initialized, fetch state etc.

The DML statement I’m using in this example is inserting a row in two columns table.

{% highlight sql %}
INSERT INTO t" + "(N1, V1) VALUES" + "(?,?)
{% endhighlight %}

For the insert (DML) statement another htype = statement hndl handle is allocated.
You can also see the statement executed, it’s shown at OCIStmtPrepare and OCIStmtExecute entries.

{% highlight shell %}
# 04/09/17 23:27:59:901 # Thread ID 130486272 # Entry - OCIHandleAlloc(parenth = 0x7fde708b6e00, hndlp = 0x700007c704b0, htype = statement hndl, xtramem_sz =1000, usermemp = 0x700007c70538, flag = TRUE);
# 04/09/17 23:27:59:901 # Thread ID 130486272 # Exit - OCIHandleAlloc
# 04/09/17 23:27:59:903 # Thread ID 130486272 # Entry - OCIStmtPrepare(stmtp = 0x7fde709e43d8, errhp = 0x7fde702f5a10, stmt = INSERT INTO t23(N1, V1) VALUES(:1 ,:2 ), stmt_len = 39, language = OCI_NTV_SYNTAX,  mode = OCI_DEFAULT(000000000));
# 04/09/17 23:27:59:904 # Thread ID 130486272 # Exit - OCIStmtPrepare
# 04/09/17 23:27:59:904 # Thread ID 130486272 # Entry - OCIAttrGet(trgthndlp = 0x7fde709e43d8, trghndltyp = 4, attributep = 0x7fde709e4ac0, sizep = 0x0, attrtype = OCI_ATTR_STMT_TYPE, errhp = 0x7fde702f5a10);
# 04/09/17 23:27:59:904 # Thread ID 130486272 # Exit - OCIAttrGet
# 04/09/17 23:27:59:904 # Thread ID 130486272 # Exit - OCIAttrSet
# 04/09/17 23:27:59:904 # Thread ID 130486272 # Entry - OCIStmtExecute(svchp = 0x7fde702f5928, stmhp = 0x7fde709e43d8, errhp = 0x7fde702f5a10, iters = 1, rowoff = 0, snap_in = 0x0, snap_out = 0x0, mode = OCI_DEFAULT(000000000)for sql :
INSERT INTO t23(N1, V1) VALUES(:1 ,:2 )
# 04/09/17 23:27:59:907 # Thread ID 130486272 # Exit - OCIStmtExecute
# 04/09/17 23:27:59:907 # Thread ID 130486272 # Entry - OCIAttrGet(trgthndlp = 0x7fde709e43d8, trghndltyp = 4, attributep = 0x700007c70184, sizep = 0x0, attrtype = OCI_ATTR_ROW_COUNT, errhp = 0x7fde702f5a10);
# 04/09/17 23:27:59:907 # Thread ID 130486272 # Exit - OCIAttrGet
# 04/09/17 23:27:59:908 # Thread ID 130486272 # Entry - OCITransCommit(svchp = 0x7fde702f5928, errhp = 0x7fde702f5a10, flags = OCI_DEFAULT);
# 04/09/17 23:27:59:910 # Thread ID 130486272 # Exit - OCITransCommit
# 04/09/17 23:27:59:911 # Thread ID 130486272 # Entry - OCIHandleFree(hndlp = 0x7fde709e43d8, type = statement hndl);
# 04/09/17 23:27:59:911 # Thread ID 130486272 # Exit - OCIHandleFree
# 04/09/17 23:27:59:911 # Thread ID 130486272 # Entry - OCISessionEnd(svchp = 0x7fde702f5928, errhp = 0x7fde702f5a10, usrhp = 0x7fde7119f380,  : user=******,mode = OCI_DEFAULT(000000000)# 04/09/17 23:27:59:914 # Thread ID 130486272 # Exit - OCISessionEnd
# 04/09/17 23:27:59:915 # Thread ID 130486272 # Entry - OCIServerDetach(srvhp = 0x7fde701bfc60, errhp = 0x7fde702f5a10, mode = OCI_DEFAULT);
(000000000)# 04/09/17 23:27:59:916 # Thread ID 130486272 # Exit - OCIServerDetach
# 04/09/17 23:27:59:916 # Thread ID 130486272 # Entry - OCIHandleFree(hndlp = 0x7fde701bfc60, type = server hndl);
# 04/09/17 23:27:59:917 # Thread ID 130486272 # Exit - OCIHandleFree
# 04/09/17 23:27:59:917 # Thread ID 130486272 # Entry - OCIHandleFree(hndlp = 0x7fde7119f380, type = authentication hndl);
# 04/09/17 23:27:59:917 # Thread ID 130486272 # Exit - OCIHandleFree
# 04/09/17 23:27:59:917 # Thread ID 130486272 # Entry - OCIHandleFree(hndlp = 0x7fde702f5928, type = service hndl);
# 04/09/17 23:27:59:917 # Thread ID 130486272 # Exit - OCIHandleFree
# 04/09/17 23:27:59:917 # Thread ID 130486272 # Entry - OCIHandleFree(hndlp = 0x7fde702f5a10, type = error hndl);
# 04/09/17 23:27:59:917 # Thread ID 130486272 # Exit - OCIHandleFree
# 04/09/17 23:27:59:917 # Thread ID 130486272 # Entry - OCIHandleFree(hndlp = 0x7fde708b6e00, type = environment hndl);
{% endhighlight %}

The first step is for the statement to be prepared for execution, this step is executed under OCIStmtPrepare function call.
stmt = INSERT INTO t23(N1, V1) VALUES(:1 ,:2 ) – the actual statement
stmt_len = 39 – the length of the statement
language = OCI_NTV_SYNTAX – the language type for syntax check

On line 5, OCIAttrGet is called to identify the statement type via the attribute OCI_ATTR_STMT_TYPE, in this case it’s OCI_STMT_INSERT.

The statement is executed via OCIStmtExecute, line 8.
After execution, the OCI_ATTR_ROW_COUNT attribute is retrieved which stands for “Number of rows successfully converted in the last call to”. The same see with SQL%ROWCOUNT.

The transaction is committed with OCITransCommit function call. In this case the default flags = OCI_DEFAULT is used, for one-phase commit since the transaction is non-distributed.

After the statement is executed several OCI function calls are called (OCISessionEnd and OCIServerDetach) to end the session and detach the connection. Followed by several OCIHandleFree calls to clean several previously allocated handles.

Also for each connection there is an error handle allocated which is used to record information if an error occurs. Let’s simulate this with an insert statement which will fail with ORA-01722: invalid number

{% highlight shell %}
# 04/10/17 18:34:51:187 # Thread ID 3292144576 # Entry - OCIErrorGet(hndlp = 0x7fe202010c10, recordno = 1, sqlstate = NULL, errcodep = 0x7fff5ac20c08 = 0, bufsiz = 2048, type = OCI_HTYPE_ERROR);
# 04/10/17 18:34:51:187 # Thread ID 3292144576 # Exit - OCIErrorGet
# 04/10/17 18:34:51:188 # Thread ID 3292144576 # Entry - OCIErrorGet(hndlp = 0x7fe202010c10, recordno = 2, sqlstate = NULL, errcodep = 0x7fff5ac20c08 = 1722, bufsiz = 2048, type = OCI_HTYPE_ERROR);
# 04/10/17 18:34:51:188 # Thread ID 3292144576 # Exit - OCIErrorGet
{% endhighlight %}

The error code is reported via OCIErrorGet function call, in this case errcodep = 1722.
The statement which was executed:

{% highlight sql %}
SQL> insert into t  values   ('iarsov','test');
insert into t  values   ('iarsov','test')
*
ERROR at line 1:
ORA-01722: invalid number
{% endhighlight %}
  
For a SELECT statement there are multiple OCIStmtFetch calls, depending on the array size. I’ve used default array size of 15 and which is noted in nrows.

{% highlight shell %}
# 04/10/17 18:51:46:868 # Thread ID 3292144576 # Entry - OCIStmtPrepare(stmtp = 0x7f98e38058f8, errhp = 0x7f98e2853210, stmt = select * from t, stmt_len = 17, language = OCI_NTV_SYNTAX,  mode = 32(0x0000020));
# 04/10/17 18:51:46:868 # Thread ID 3292144576 # Exit - OCIStmtPrepare
# 04/10/17 18:51:46:868 # Thread ID 3292144576 # Exit - OCIStmtPrepare2(stmhp = 0x7f98e38058f8);
# 04/10/17 18:51:46:868 # Thread ID 3292144576 # Exit - OCIStmtGetBindInfo
# 04/10/17 18:51:46:868 # Thread ID 3292144576 # Entry - OCIStmtExecute(svchp = 0x7f98e2854d28, stmhp = 0x7f98e38058f8, errhp = 0x7f98e2853210, iters = 0, rowoff = 0, snap_in = 0x0, snap_out = 0x0, mode = OCI_DEFAULT(000000000)for sql :
select * from t23
# 04/10/17 18:51:46:883 # Thread ID 3292144576 # Exit - OCIStmtExecute
# 04/10/17 18:51:46:884 # Thread ID 3292144576 # Entry - OCIDescriptorAlloc(parenth = 0x7f98e38058f8, descpp = 0x7fff52a1d4f8, trc_type = 53 = parameter descriptor, xtramem_sz = 0, usrmempp = 0x0);
# 04/10/17 18:51:46:884 # Thread ID 3292144576 # Exit - OCIDescriptorAlloc
# 04/10/17 18:51:46:885 # Thread ID 3292144576 # Entry - OCIDescriptorAlloc(parenth = 0x7f98e38058f8, descpp = 0x7fff52a1d4f8, trc_type = 53 = parameter descriptor, xtramem_sz = 0, usrmempp = 0x0);
# 04/10/17 18:51:46:885 # Thread ID 3292144576 # Exit - OCIDescriptorAlloc
# 04/10/17 18:51:46:886 # Thread ID 3292144576 # Entry - OCIStmtFetch(stmhp = 0x7f98e38058f8, errhp = 0x7f98e2853210, nrows = 15, orientation = OCI_FETCH_NEXT, mode = OCI_DEFAULT(000000000)for sql :
select * from t
));
# 04/10/17 18:51:46:887 # Thread ID 3292144576 # Exit - OCIStmtFetch
# 04/10/17 18:51:46:889 # Thread ID 3292144576 # Entry - OCIStmtFetch(stmhp = 0x7f98e38058f8, errhp = 0x7f98e2853210, nrows = 15, orientation = OCI_FETCH_NEXT, mode = OCI_DEFAULT(000000000)for sql :
select * from t
));
# 04/10/17 18:51:46:891 # Thread ID 3292144576 # Exit - OCIStmtFetch
# 04/10/17 18:51:46:891 # Thread ID 3292144576 # Entry - OCIStmtFetch(stmhp = 0x7f98e38058f8, errhp = 0x7f98e2853210, nrows = 15, orientation = OCI_FETCH_NEXT, mode = OCI_DEFAULT(000000000)for sql :
select * from t
));
# 04/10/17 18:51:46:891 # Thread ID 3292144576 # Exit - OCIStmtFetch
# 04/10/17 18:51:46:891 # Thread ID 3292144576 # Entry - OCIStmtRelease(stmtp = 0x7f98e38058f8, errhp = 0x7f98e2853210, key = , key_len = 0, mode = OCI_DEFAULT# 04/10/17 18:51:46:891 # Thread ID 3292144576 # Exit - OCIStmtRelease
# 04/10/17 18:51:46:891 # Thread ID 3292144576 # Entry - OCIHandleFree(hndlp = 0x7f98e38058f8, type = statement hndl);
{% endhighlight %}

It’s not likely that you will need to perform OCI calls tracing on a daily basis.
However, it can be useful in troubleshooting some issues at OCI call level.