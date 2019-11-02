---
layout: post
title:  "Date And Time Format In ASMCMD For File Listing"
date:   2018-05-11
categories: oracle
---

This is a short blog post to give you an idea how you can define your own custom date and time format for "ls -l" command in ASMCMD. If you use ASMCMD quite often, you might have been (at least once) in a situation where you wanted to format the date and time of "ls -l" command, but you did not find a way to do it.

If you did not know, in ASMCMD utility the date and time format for "ls -l" command is hardcoded. Possible formats are "MON DD YYYY" or "MON DD HH24:MI:SS", based on file age. With RMAN we can use NLS_DATE_FORMAT environment variable to define our desired format, but that’s not the case with ASMCMD.

ASMCMD is a Perl utility that provides navigation of directories/files within ASM disk groups. Its modules are stored in $CRS_HOME/lib directory.
ASMCMD utility base module is **asmcmdbase.pm**.

The subroutine for listing files located in the base module is **asmcmdbase_ls_process_file**. You can see the logic how file dates are printed. We can notice that if the file is older than 6 months the format used is "MON DD YYYY". If the file is newer, the format used is "MON DD HH24:MI:SS". This subroutine is called (executed) for each file that needs to be printed in ASMCMD.
List entries _$entry_info_ref->{'mod_date'}_ and _$entry_info_ref->{'mod_time'}_ contain the data for both hardcoded formats. MOD_DATE if the files is older than 6 months and MOD_TIME if the file is newer.

{% highlight bash %}
...

sub asmcmdbase_ls_process_file
{
my ($dbh, $entry_info_ref, $args_ref, $cur_date, $file_info) = @_;

my ($related_alias); # The user or system alias for its corresponding #

# system or user alias, respectively.

# Calculate dates only if we have file info from v$asm_file.

if ($file_info)
{

# Use separate date format if date older than half a year.

if (($cur_date - $entry_info_ref->{'julian_date'}) >= $ASMCMDBASE_SIXMONTH)
{ # If older than six months, use 'MON DD YYYY' format. #
$entry_info_ref->{'date_print'} = $entry_info_ref->{'mod_date'};
}
else
{ # Otherwise, use 'MON DD HH24:MI:SS' format. #
$entry_info_ref->{'date_print'} = $entry_info_ref->{'mod_time'};
}
}

...
{% endhighlight %}

In order to understand how _$entry_info_ref->{'mod_date'}_ and _$entry_info_ref->{'mod_date'}_ are defined we need to look at **asmcmdshare.pm** module which is the ASM command line interface (Shared Functionality Module).

Within this module, in subroutine asmcmdshare_get_file we can find the actual query statement used to get file information from V$ASM_FILE. We see that date format is hardcoded within the select statement.
Control ASMCMD date time format with environment variable.

By now, you’ve probably figure it out what you need to do in order to get your custom format in place. E.g. if you want to control ASMCMD date format with OS environment variable you just need very few lines of code.

Let’s assume we want to control the format with NLS_DATE_FORMAT_XX environemnt variable.

1. Add condition with default format to be used if environment variable is not set. In our case that would be 'DD.MM.YYYY HH24:MI:SS'
2. Add additional column in the query to get the date formatted based on the environment variable definition.

{% highlight bash %}
...

if(!$ENV{'NLS_DATE_FORMAT_XX'}){ $ENV{'NLS_DATE_FORMAT_XX'} = "dd.mm.yyyy hh24:mi:ss"; }

$qry = 'select group_number, file_number, incarnation, block_size, ' .
'blocks, bytes, space, type, redundancy, striped, creation_date, ' .
'user_number, usergroup_number, permissions, ' .
'to_char(modification_date, 'MON DD HH24:MI:SS') "MOD_TIME", ' .
'to_char(modification_date, ''.$ENV{'NLS_DATE_FORMAT_XX'}.'') "MOD_TIME_XX", ' .
'to_char(modification_date, 'MON DD YYYY') "MOD_DATE", ' .
'to_char(modification_date, 'J HH24 MI SS') "JULIAN_TIME", ' .
'to_char(modification_date, 'J') "JULIAN_DATE" ' .
'from v$asm_file where compound_index = ? ORDER BY ' .
'GROUP_NUMBER, FILE_NUMBER, USER_NUMBER, USERGROUP_NUMBER '
;

...
{% endhighlight %}

Below the select statement you need to fetch the column into $file_info_ref under your own key.

{% highlight bash %}
...

$file_info_ref->{'bytes'} = $row->{'BYTES'};
$file_info_ref->{'space'} = $row->{'SPACE'};
$file_info_ref->{'type'} = $row->{'TYPE'};
$file_info_ref->{'redundancy'} = $row->{'REDUNDANCY'};
$file_info_ref->{'striped'} = $row->{'STRIPED'};
$file_info_ref->{'creation_date'} = $row->{'CREATION_DATE'};
$file_info_ref->{'mod_time'} = $row->{'MOD_TIME'};
$file_info_ref->{'mod_time_xx'} = $row->{'MOD_TIME_XX'};
$file_info_ref->{'julian_time'} = $row->{'JULIAN_TIME'};
$file_info_ref->{'julian_date'} = $row->{'JULIAN_DATE'};
$file_info_ref->{'user_number'} = $row->{'USER_NUMBER'};

...
{% endhighlight %}

As final step, in the main ASMCMD module, subroutine _asmcmdbase_ls_process_file_ you need to overwrite _$entry_info_ref->{'date_print'}_ to be populated with previously defined custom key which holds the data formatted as per 'NLS_DATE_FORMAT_XX'.

{% highlight bash %}
...

# Calculate dates only if we have file info from v$asm_file.

if ($file_info)
{

# Use separate date format if date older than half a year.

if (($cur_date - $entry_info_ref->{'julian_date'}) >= $ASMCMDBASE_SIXMONTH)
{ # If older than six months, use 'MON DD YYYY' format. #
$entry_info_ref->{'date_print'} = $entry_info_ref->{'mod_date'};
}
else
{ # Otherwise, use 'MON DD HH24:MI:SS' format. #
$entry_info_ref->{'date_print'} = $entry_info_ref->{'mod_time'};
}

# Overwrite date_print

entry_info_ref->{'date_print'} = $entry_info_ref->{'mod_time_xx'};

}

...
{% endhighlight %}

Now you can just do the following.

{% highlight bash %}
[oracle@host01 lib]$ export NLS_DATE_FORMAT_XX="yyyy.mm.dd hh24:mi:ss"
[oracle@host01 lib]$
[oracle@host01 lib]$ asmcmd
ASMCMD> ls -l data/orcl/spfileorcl.ora
Type Redund Striped Time Sys Name
PARAMETERFILE UNPROT COARSE 2018.03.09 02:00:00 N spfileorcl.ora => +DATA/ORCL/PARAMETERFILE/spfile.267.965812191
ASMCMD>
ASMCMD>
ASMCMD> exit
[oracle@host01 lib]$
[oracle@host01 lib]$ export NLS_DATE_FORMAT_XX="dd.mm.yyyy hh24:mi:ss"
[oracle@host01 lib]$
[oracle@host01 lib]$ asmcmd
ASMCMD>
ASMCMD> ls -l data/orcl/spfileorcl.ora
Type Redund Striped Time Sys Name
PARAMETERFILE UNPROT COARSE 09.03.2018 02:00:00 N spfileorcl.ora => +DATA/ORCL/PARAMETERFILE/spfile.267.965812191
{% endhighlight %}

Use it at your own risk.
