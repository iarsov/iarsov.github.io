

Oracle with the release of 12c introduced new feature called invisible columns which basically allows developers to create new columns (as I would say) not visible to the world 🙂 . It enables end-users to continue accessing the application while developers enhancing the application.
These kind of columns are not visible to the application because in order to access the column we have to explicitly specify the column name in the select statement. “SELECT * FROM …” will also not show the column. This is because COLUMN_ID column in _TAB_COLUMNS dictionary views which is used to determine the order of columns retrieved when you run ” SELECT * FROM …” has a NULL value.

How do you define invisible columns?
It is very straightforward with the INVISIBLE keyword.

{% highlight sql %}
SQL> create table mytable(id number, col1 number invisible);

Table created.
{% endhighlight %}

If I describe previously created table we won’t see the columns that are set invisible.

{% highlight sql %}
SQL> desc mytable
 Name              Null?    Type
 ----------------- -------- ------------
 ID                         NUMBER
{% endhighlight %}

Of course there is a solution for that.
We can set COLINVISIBLE command in sqlplus, lets try.

{% highlight sql %}
SQL> set colinvisible on
SQL>
SQL> desc mytable
 Name                      Null?    Type
 ------------------------- -------- ------------
 ID                                 NUMBER
 COL1 (INVISIBLE)                   NUMBER
{% endhighlight %}

In order to check which columns are invisible we have to search NULL value for COLUMN_ID in *_USER_TAB_COLUMNS dictionary views.

{% highlight sql %}
SQL> col table_name format a10
SQL> col column_name format a10
SQL>
SQL> select table_name,column_name 
from user_tab_columns 
where table_name = 'MYTABLE' and column_id is null;

TABLE_NAME COLUMN_NAME
---------- -----------
MYTABLE    COL1
{% endhighlight %}

When we’re going to make the column visible the system will generate new highest number for COLUMN_ID column.

{% highlight sql %}
SQL> select table_name,column_name,column_id 
from user_tab_columns 
where table_name = 'MYTABLE' 
order by column_id;

TABLE_NAME COLUMN_NAME  COLUMN_ID
---------- ----------- ----------
MYTABLE    ID                  1
MYTABLE    COL1                2
{% endhighlight %}

We can also specify INVISIBLE on virtual columns.

{% highlight sql %}
SQL> alter table mytable add col2 as (col1 * 10);

Table altered.

SQL>
SQL> alter table mytable modify col2 invisible;

Table altered.

SQL>
SQL> desc mytable;
 Name                                      Null?    Type
 ----------------------------------------- -------- ---------------
 ID                                                 NUMBER
 COL1                                               NUMBER
 COL2 (INVISIBLE)                                   NUMBER
{% endhighlight %}

Lets insert sample data and see the effect.

{% highlight sql %}
SQL> select id, col1, col2 from mytable;

no rows selected

SQL> insert into mytable values (1,10);

1 row created.

SQL> commit;

Commit complete.

SQL> select * from mytable;

        ID       COL1
---------- ----------
         1         10
{% endhighlight %}

As you can see we didn’t got COL2 column because it’s set to INVISIBLE.
Lets specify the column in the SELECT statement.

{% highlight sql %}
SQL> select id, col1, col2 from mytable;

        ID       COL1       COL2
---------- ---------- ----------
         1         10        100
{% endhighlight %}

The restrictions for this feature is that you can’t use it on temp, external or cluster tables.
You can’t also make system-generated invisible columns visible.
