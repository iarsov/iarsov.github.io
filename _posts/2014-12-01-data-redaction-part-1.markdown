---
layout: post
title:  "Data Redaction – Part 1"
date:   2014-12-01
categories: oracle
---

If you use application logic to mask sensitive data now with 12c you have an option to accomplish the same effect at database level, but more efficient and secure. This is called Oracle Data Redaction. The solution is very simple, it modifies sensitive data based on user-defined policies. Important note to remember is that the policies that are defined does not modify data blocks at storage level. Instead, the data is modified on-the-fly before the results are returned to the user/application. This represents very fast and easy to implement solution for data masking.
We can mask characters, numbers and dates. Before redaction to take place we need to define policies which will determine the type of redaction, what to redact and how to redact.

There are several types from which we can choose: NONE , FULL, PARTIAL, RANDOM and REGEXP.

NONE – masking is not used<br/>
FULL – data will be replaced with fixed value<br/>
PARTIAL – we define what to be masked<br/>
RANDOM – the system will generate random value which will be used to replace original data<br/>
REGEXP – data is masked based on regular expression pattern

In this part 1 I am going to show data redaction for characters, in the following part I’ll write for numbers and dates.

***Preparation***

I have created the following table just for the purpose of this post (I haven’t included any architectural design principals).

{% highlight sql %}
SQL> conn iarsov@pdbx
Enter password:
Connected.

SQL> create table mtab(user_id number,
name varchar2(20),
surname varchar2(50),
account varchar2(15),
created date);

Table created.

SQL>
SQL> insert into mtab values(1, 'Steven', 'King', '123-4567-891234', sysdate);

1 row created.

SQL>
SQL> select user_id,name,surname,account,created from mtab;

   USER_ID NAME   SURNAM ACCOUNT         CREATED
---------- ------ ------ --------------- ----------
         1 Steven King   123-4567-891234 24.11.2014
{% endhighlight %}

Lets define policy to redact/mask account column. The package that we need is called DBMS_REDACT.

We add policy with ADD_POLICY procedure. Required parameters are object_name, policy_name and expression. All other parameters have default values. For object_name we need to pass the object for which we want to create the policy, this can be table or view. For expression we need to specify expression that needs to evaluate to boolean value, if the expression evaluates to TRUE then the policy will be used. In this example I’ve set expression so that this policy is used for all users except “IARSOV” user.

**FULL data redaction**

For FULL type the system will use default values. We can determine which default values are used by querying REDACTION_VALUES_FOR_TYPE_FULL dictionary view. Also, those values can be changed/updated with UPDATE_FULL_REDACTION_VALUES procedure.

I’ve created the following policy for mtab table and account column with FULL type.

{% highlight sql %}
SQL> conn iarsov@pdbx
Enter password:
Connected.

SQL> @redact_full.sql
SQL>
SQL> begin
  2
  3  DBMS_REDACT.ADD_POLICY (
  4     object_schema => USER,
  5     object_name => 'mtab',
  6     policy_name => 'mtab_account_full',
  7     column_name => 'ACCOUNT',
  8     function_type => DBMS_REDACT.FULL,
  9     function_parameters => null,
 10     expression => 'SYS_CONTEXT(''USERENV'',''CURRENT_USER'') <> ''IARSOV''');
 11  end;
 12  /

PL/SQL procedure successfully completed.
{% endhighlight %}

In order to test previously created policy I’ve created another user orax which has been granted SELECT privilege for iarsov.mtab table.

{% highlight sql %}
SQL> conn system@pdbx
Enter password:
Connected.

SQL>
SQL>
SQL> create user orax identified by orax;

User created.

SQL> grant create session to orax;

Grant succeeded.

SQL> grant select on iarsov.mtab to orax;

Grant succeeded.

SQL> conn orax@pdbx
Enter password:
Connected.

SQL> select name,surname,account from iarsov.mtab;

NAME   SURNAM ACCOUNT
-----  ------ ---------------
Steven King
{% endhighlight %}

The value for account column has been replaced with empty string which is default value for FULL data redaction.

**PARTIAL data redaction**

For PARTIAL type we have to define the function_parameters parameter which is used for masking. This parameter is constructed from several fields: input format, output format, mask character, start position and end position.
Input format accepts two values ‘V’ and ‘F’.

‘V’ serve to mark values which can be possibly masked.
‘F’ serve to mark characters to be ignored.

In this example I have account number which has the following format nnn-nnnn-nnnnnn (where n is number), with value 123-4567-891234. I would like to mask that value to look like xxx-xxxx-xxx234.

The expression that I need is: ‘VVVFVVVVFVVVVVV,VVV-VVVV-VVVVVV,x,1,10’ where:<br/>
‘VVVFVVVVFVVVVVV’ is input format<br/>
‘VVV-VVVV-VVVVVV’ is output format<br/>
‘x’ is mask character<br/>
‘1’ is start position<br/>
’10’ is end position

What we need to notice is that ‘F’ characters are not taken into consideration when determine the start position and end position.

{% highlight sql %}
SQL> conn iarsov@pdbx
Enter password:
Connected.

SQL>
SQL> @redact_partial.sql
SQL>
SQL> begin
 2
 3 DBMS_REDACT.ADD_POLICY (
 4 object_schema => USER,
 5 object_name => 'mtab',
 6 policy_name => 'mtab_account_partial',
 7 column_name => 'ACCOUNT',
 8 function_type => DBMS_REDACT.PARTIAL,
 9 function_parameters => 'VVVFVVVVFVVVVVV,VVV-VVVV-VVVVVV,x,1,10',
 10 expression => 'SYS_CONTEXT(''USERENV'',''CURRENT_USER'') <> ''IARSOV''');
 11
 12 end;
 13 /

PL/SQL procedure successfully completed.
{% endhighlight %}

Lets check if orax user sees the account with masked value.

{% highlight sql %}
SQL> conn orax@pdbx
Enter password:
Connected.

SQL>
SQL> select name,surname,account from iarsov.mtab;

NAME   SURNAM ACCOUNT
-----  ------ ---------------
Steven King   xxx-xxxx-xxx234
{% endhighlight %}

**RANDOM data redaction**

This type is very straightforward. What we need is just to create the policy without to specify value for function_parameters parameter.

{% highlight sql %}
SQL> conn iarsov@pdbx
Enter password:
Connected.

SQL> exec dbms_redact.drop_policy(user,'mtab','mtab_account_partial');

PL/SQL procedure successfully completed.

SQL>
SQL> @redact_random.sql
SQL>
SQL> begin
 2
 3 DBMS_REDACT.ADD_POLICY (
 4 object_schema => USER,
 5 object_name => 'mtab',
 6 policy_name => 'mtab_account_random',
 7 column_name => 'ACCOUNT',
 8 function_type => DBMS_REDACT.RANDOM,
 9 function_parameters => null,
 10 expression => 'SYS_CONTEXT(''USERENV'',''CURRENT_USER'') <> ''IARSOV''');
 11 end;
 12 /

PL/SQL procedure successfully completed.
{% endhighlight %}

Lets confirm that our policy is working.

{% highlight sql %}
SQL> conn orax@pdbx
Enter password:
Connected.

SQL>
SQL> select name,surname,account from iarsov.mtab;

NAME   SURNAM ACCOUNT
-----  ------ ---------------
Steven King   3>(`<P}{;F0hDX&

SQL> /

NAME   SURNAM  ACCOUNT
-----  ------  ---------------
Steven King    1!p6i3qM9)x>q;y
{% endhighlight %}

As you can see we got random characters. We can note that on the second execution of the query we got different random characters.
This is confirms that masking is taking place on the fly and no data blocks are modified.

**REGEXP redact**

We can use this type for special cases when we can’t create policy based on other types. The important note for this type is that it’s based on regular expressions.
For example lets mask the account value to same format as we used for PARTIAL type, but this time we’re going to use regular expression.

The regular expression that we need is ‘([[:digit:]]{3})-([[:digit:]]{4})-([[:digit:]]{3})’.

{% highlight sql %}
SQL> conn iarsov@pdbx
Enter password:
Connected.

SQL> @redact_partial_regex.sql
SQL>
SQL> begin
 2
 3 DBMS_REDACT.ADD_POLICY (
 4 object_schema => USER,
 5 object_name => 'mtab',
 6 policy_name => 'mtab_account_partial',
 7 column_name => 'ACCOUNT',
 8 function_type => DBMS_REDACT.REGEXP,
 9 function_parameters => null,
 10 expression => 'SYS_CONTEXT(''USERENV'',''CURRENT_USER'') <> ''IARSOV''',
 11 regexp_pattern => '([[:digit:]]{3})-([[:digit:]]{4})-([[:digit:]]{3})',
 12 regexp_replace_string => 'xxx-xxxx-xxx');
 13
 14 end;
 15 /

PL/SQL procedure successfully completed.
{% endhighlight %}

Here to remember is that when we’re using REGEXP we must not use _function_parameters_ parameter.
Lets confirm that our regular expression is correct and the policy is working.

{% highlight sql %}
SQL> conn orax@pdbx
Enter password:
Connected.

SQL>
SQL> select name,surname,account from iarsov.mtab;

NAME   SURNAME ACCOUNT
-----  ------- ---------------
Steven King    xxx-xxxx-xxx234
{% endhighlight %}

We can also exempt some users from this data redact policy restrictions by granting EXEMPT REDACTION POLICY system privilege to the users.

{% highlight sql %}
SQL> conn system@pdbx
Enter password:
Connected.

SQL>
SQL> grant exempt redaction policy to orax;

Grant succeeded.
{% endhighlight %}

Confirm that now orax user can see account data from iarsov.mtab table.

{% highlight sql %}
SQL> conn orax@pdbx
Enter password:
Connected.

SQL>
SQL> select name,surname,account from iarsov.mtab;

NAME   SURNAME ACCOUNT
-----  ------- ---------------
Steven King    123-4567-891234
{% endhighlight %}

There is also another system privilege EXEMPT DDL REDACTION POLICY that exempt users from data redaction when they’re performing DDL operation.