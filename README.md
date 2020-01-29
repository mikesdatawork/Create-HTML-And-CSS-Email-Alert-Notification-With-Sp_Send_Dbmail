![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_01.png "Mikes Data Work")        

# Create HTML And CSS Email Alert Notification With Sp_Send_Dbmail
**Post Date: July 1, 2015**        



## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)       

## About-Process

<p>Here's a process I created to email a fairly nice HTML & CSS formatted email (not using attachments) which displays Job Error, and includes the Step name, and error message. It will look like this:

![SQL HTML Email Example]( https://mikesdatawork.files.wordpress.com/2015/07/image0011.jpg "SQL Alert")
 
What This will do is create a couple temporary tables to hold the Agent Job History information, then pass that information into the sp_send_dbmail using CSS and HTML formatting.
Special thanks to Bru Medishetty at MSSQLTIPS for his original write up for "Send email in a tabular format using SQL Server database mail" which used for some reference syntax. His original write-up can be found here: http://www.mssqltips.com/sqlservertip/2347/send-email-in-a-tabular-format-using-sql-server-database-mail/</p>    



## SQL-Logic
```SQL
use msdb;
set nocount on
 
-- find most recent error step error in sysjobhistory and pull name based on instance_id from history table.
declare @last_error                      varchar(255)
declare       @last_error_job_name varchar(255)
set           @last_error                       = ( select top 1 instance_id from sysjobhistory where message like '%The step failed%' order by run_date desc )
set           @last_error_job_name = ( select sj.name from sysjobs sj join sysjobhistory sjh on sj.job_id = sjh.job_id where instance_id = @last_error )
 
-- create temp table to store error information.
if object_id('tempdb..#agent_job_step_error_report') is not null
       drop table #agent_job_step_error_report
 
create table #agent_job_step_error_report
(
       [id]                       int identity (1,1)
,      [server_name]        varchar(255)
,      [time_of_error]            varchar(255)
,      [job_name]                 varchar(255)
,      [step_id]                  varchar(255)
,      [step_name]                varchar(255)
,      [duration]                 varchar(255)
,      [error_message]            varchar(max)
)
 
-- get information from job system tables for job step error report.
insert into #agent_job_step_error_report ([server_name], [time_of_error], [job_name], [step_id], [step_name], [duration], [error_message])
select
       'server name'        = @@servername
,      'time of error'            = datename(dw, msdb.dbo.agent_datetime(run_date, run_time) ) + ':  ' + convert(char, msdb.dbo.agent_datetime(run_date, run_time) , 9)
,      'job name'                 = sj.name
,      'step id'                  = sjh.step_id
,      'step name'                = sjh.step_name
,      'duration'                 = CAST(sjh.run_duration/10000 as varchar)  + ':' + CAST(sjh.run_duration/100%100 as varchar) + ':' + CAST(sjh.run_duration%100 as varchar)
,      'error message'            = sjh.message
from
       msdb..sysjobs sj join msdb..sysjobhistory sjh on sj.job_id = sjh.job_id
where
       instance_id = @last_error
 
 
-- crate temp table to store job information
if object_id('tempdb..#agent_job_information') is not null
       drop table #agent_job_information
 
create table #agent_job_information
(
       [id]                 int identity (1,1)
,      [job_name]           varchar(255)
,      [step_id]            varchar(255)
,      [step_name]          varchar(255)
,      [process_type]       varchar(255)
,      [previous_run]       varchar(255)
)
 
-- get all job, and step info for quick reference including the previous run duration and the last known run timestamp before the most previous error.
insert into #agent_job_information ([job_name], [step_id], [step_name], [process_type], [previous_run])
select
       'job name'                        = sj.name
,      'step id'                         = sjs.step_id
,      'step name'                       = sjs.step_name
,      'process type'                    = sjs.subsystem
,      'previous run'                    = datename(dw, dateadd(millisecond, sjs.last_run_time,convert(datetime,cast(nullif(sjs.last_run_date,0) as nvarchar(10))))) + ':  ' + convert(char, dateadd(millisecond, sjs.last_run_time,convert(datetime,cast(nullif(sjs.last_run_date,0) as nvarchar(10)))), 9)
from
       msdb..sysjobs sj join msdb..sysjobsteps sjs on sj.job_id = sjs.job_id
where
       sj.name = @last_error_job_name
order by
       sj.name, sjs.step_id asc
 
-- create html tables top
declare @xml_top NVARCHAR(MAX)
declare @body_top NVARCHAR(MAX)
 
 
set @xml_top = 
       cast(
              (select
                     [server_name] as 'td'
              ,      ''
              ,      [time_of_error]      as 'td'
              ,      ''
              ,      [job_name]           as 'td'
              ,      ''
              ,      [step_id]            as 'td'
              ,      ''
              ,      [step_name]          as 'td'
              ,      ''
              ,      [duration]           as 'td'
              ,      ''
              ,      [error_message]      as 'td'
              from  #agent_job_step_error_report 
              --order by rank 
              for xml path('tr')
       ,      elements)
       as NVARCHAR(MAX)
              )
 
set @body_top =
              '&amp;amp;lt;html&amp;amp;gt;
              &amp;amp;lt;head&amp;amp;gt;
                     &amp;amp;lt;style&amp;amp;gt;
                                  h3{
                                         font-family: sans-serif;
                                         color: red;
                                  }
 
                                  table, td, tr, th {
                                         font-family: sans-serif;
                                         border: 1px solid black;
                                         border-collapse: collapse;
                                  }
                                  th {
                                         text-align: left;
                                         background-color: gray;
                                         color: white;
                                         padding: 5px;
                                  }
 
                                  td {
                                         padding: 5px;
                                  }
 
                                   
                     &amp;amp;lt;/style&amp;amp;gt;
              &amp;amp;lt;/head&amp;amp;gt;
              &amp;amp;lt;body&amp;amp;gt;
              &amp;amp;lt;H3&amp;amp;gt;SQL Agent Job Error&amp;amp;lt;/H3&amp;amp;gt;
              &amp;amp;lt;table border = 1&amp;amp;gt;
              &amp;amp;lt;tr&amp;amp;gt;
                     &amp;amp;lt;th&amp;amp;gt; Server Name     &amp;amp;lt;/th&amp;amp;gt;
                     &amp;amp;lt;th&amp;amp;gt; Time of Error   &amp;amp;lt;/th&amp;amp;gt;
                     &amp;amp;lt;th&amp;amp;gt; Job Name        &amp;amp;lt;/th&amp;amp;gt;
                     &amp;amp;lt;th&amp;amp;gt; Step ID         &amp;amp;lt;/th&amp;amp;gt;
                     &amp;amp;lt;th&amp;amp;gt; Step Name             &amp;amp;lt;/th&amp;amp;gt;
                     &amp;amp;lt;th&amp;amp;gt; Duration        &amp;amp;lt;/th&amp;amp;gt;
                     &amp;amp;lt;th&amp;amp;gt; Error Message   &amp;amp;lt;/th&amp;amp;gt;
              &amp;amp;lt;/tr&amp;amp;gt;'
 
set @body_top = @body_top + @xml_top + '&amp;amp;lt;/table&amp;amp;gt;&amp;amp;lt;/body&amp;amp;gt;&amp;amp;lt;/html&amp;amp;gt;'
 
 
/*
-- create html tables mid
declare @xml_mid NVARCHAR(MAX)
declare @body_mid NVARCHAR(MAX)
 
 
set @xml_mid = 
       cast(
              (select 
                     [job_name]           as 'td'
              ,      ''
              ,      [step_id]            as 'td'
              ,      ''
              ,      [step_name]          as 'td'
              ,      ''
              ,      [process_type]       as 'td'
              ,      ''
              ,      [previous_run]       as 'td'
 
              from  #agent_job_information 
              order by [job_name], [step_id] asc 
              for xml path('tr')
       ,      elements)
       as NVARCHAR(MAX)
              )
 
set @body_mid =
              '&amp;amp;lt;html&amp;amp;gt;
              &amp;amp;lt;head&amp;amp;gt;
                     &amp;amp;lt;style&amp;amp;gt;
                                  H3{
                                         font-family: sans-serif;
                                         color: red;
                                  }
 
                                  table, td, tr, th {
                                         font-family: sans-serif;
                                         border: 1px solid black;
                                         border-collapse: collapse;
                                  }
                                   
                     &amp;amp;lt;/style&amp;amp;gt;
              &amp;amp;lt;/head&amp;amp;gt;
              &amp;amp;lt;body&amp;amp;gt;
              &amp;amp;lt;H3&amp;amp;gt;About Job&amp;amp;lt;/H3&amp;amp;gt;
              &amp;amp;lt;table border = 1&amp;amp;gt;
              &amp;amp;lt;tr&amp;amp;gt;
                     &amp;amp;lt;th&amp;amp;gt; Job Name        &amp;amp;lt;/th&amp;amp;gt;
                     &amp;amp;lt;th&amp;amp;gt; Step ID         &amp;amp;lt;/th&amp;amp;gt;
                     &amp;amp;lt;th&amp;amp;gt; Step Name             &amp;amp;lt;/th&amp;amp;gt;
                     &amp;amp;lt;th&amp;amp;gt; Process Type    &amp;amp;lt;/th&amp;amp;gt;
                     &amp;amp;lt;th&amp;amp;gt; Previous Run    &amp;amp;lt;/th&amp;amp;gt;
              &amp;amp;lt;/tr&amp;amp;gt;'
 
set @body_mid = @body_mid + @xml_mid + '&amp;amp;lt;/table&amp;amp;gt;&amp;amp;lt;/body&amp;amp;gt;&amp;amp;lt;/html&amp;amp;gt;'
*/
--declare     @job_error_report    varchar(255)
--set         @job_error_report = @body_top + + @body_mid
 
EXEC msdb.dbo.sp_send_dbmail
       @profile_name        = 'SQLDatabaseMailProfile'
,      @recipients                =  Distribution groups always preferred here.'MyEmailAddress or Distribution Group'  
,      @subject                   = 'Database Mail Subject... E-mail in Tabular Format'
,      @body                      = @body_top
,      @body_format         = 'HTML';
 
drop table #agent_job_step_error_report
drop table #agent_job_information
```


[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

  
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_02.png "Mikes Data Work")

