splunk-searches
===============

Searches are what make Splunk Awesome. I thought this would be a good place to share some!

## SplunkforNagios
--


SplunkForNagios App 

Written by Luke Harris
http://verypowerful.info/home/splunkfornagios/

Downloadable from
http://apps.splunk.com/app/352
--
### Searches
--
index=main earliest=-5m | eval esize=len(_raw) | stats count max(esize) by host, source





SLA Formula 

http://en.wikipedia.org/wiki/High_availability

index=nagios src_host="rhelbuild.noc.harvard.edu" name="SSH"| head 1 | eval daysago=5 |dedup src_host,name | liveservicesla | stats max(liveservicesla) AS liveservicesla by name| eval liveservicesla=liveservicesla*100




Nagios Poor performing outliers 
1:
index=nagios sourcetype="nagioshostperf" rta=*  | stats avg(rta) as rta by src_host  | lookup local=t nagios-hostgroupmembers host_name AS src_host | sort - rta | eventstats avg(rta) as avg_rta stdev(rta) as std_rta | where rta>(2*avg_rta + std_rta) |convert ctime(_time) | rename rta as "Response Time in ms", avg_rta as "Average Response Time", std_rta as "RTA Standard Deviation", src_host as "Source Host"  

 2: ( this gives you a longer search then shows only the things that are outstanding within a shorter timeframe


index=nagios sourcetype="nagioshostperf" rta=*  | stats avg(rta) as rta by src_host, _time  | sort - rta | eventstats avg(rta) as avg_rta stdev(rta) as std_rta | where rta>(2*avg_rta + std_rta) and _time > relative_time(now(), "-35m") | rename rta as "Response Time in ms", avg_rta as "Average Response Time", std_rta as "RTA Standard Deviation", src_host as "Source Host"  





| timechart span=30m count | eventstats avg(count) as avg_count stdev(count) as std_count | where count > (avg_count + 2*std_count)

 
index=nagios sourcetype="nagioshostperf" host="nagios1-60ox.noc.harvard.edu" | stats avg(rta) as icmp_avg by src_host, _time | eventstats  stdev(icmp_avg) as std_icmp_avg  | where icmp_avg > ( std_icmp_avg * 2)  | lookup huis host as src_host | transaction src_host, bld_root


"Days since last event"
index=nagios (nagiosevent="SERVICE NOTIFICATION" ) OR (nagiosevent="HOST NOTIFICATION" ) severity=CRITICAL ( user_id=*systems*)|lookup local=t nagios-hostgroupmembers host_name AS src_host  | eval Name=coalesce(name,hostnotification) | head 1 | eval DaysSince=(now()-_time)/86400



Nagios Host Down gauge 


This Could work, and may actually work in RT, but would need a page refresh every day to input the new totals in case hosts get added/removed
`HostAlert`
index=nagios sourcetype=nagios nagiosevent="HOST ALERT"   | head 1 | livehostsdownstatus | stats max(livehostsdownstatus) AS livehostsdownstatus  | gauge livehostsdownstatus [|inputlookup alldevices.csv | fields device | stats count(device) as max | eval zed=0 |eval first=max*.25 | eval second=max*.50 | eval third=max*.75 | eval fourth=max  | eval range=zed+" "+first+" "+second+" "+third+" "+fourth | return $range ]

index=nagios sourcetype=nagios nagiosevent="HOST ALERT" hoststate=HARD |convert ctime(_time) as time | transaction startswith=DOWN endswith=UP src_host | table time, src_host, reason, name

Service Status gauge 
index=nagios sourcetype=nagios nagiosevent="HOST ALERT"   | head 1 | liveserviceokstatus | stats max(liveserviceokstatus) AS liveserviceokstatus | rangemap field=liveserviceokstatus   | gauge liveserviceokstatus [search earliest=-24h host=nagios-dev.noc.harvard.edu index="nagios" nagiosevent="CURRENT SERVICE STATE" src_host=* | stats dc(servicestatename) as services, dc(src_host) as numberofhosts by src_host | stats sum(services) as max | eval zed=0 |eval first=max*.25 | eval second=max*.50 | eval third=max*.75 | eval fourth=max  | eval range=zed+" "+first+" "+second+" "+third+" "+fourth | return $range ]

Below uses events and isn't' RT
index=nagios nagiosevent="HOST NOTIFICATION" hostnotificationstatus="DOWN"  | dedup hostnotificationstatus hostnotification | stats dc(src_host) as src_host | gauge src_host [|inputlookup alldevices.csv | fields device | stats count(device) as max | eval first=0 | eval second=max*.50 | eval third=max*.75 | eval fourth=max  | eval range=first+" "+second+" "+third+" "+fourth | return $range ]

index=nagios sourcetype=nagios nagiosevent="CURRENT HOST STATE" | head 1 | livehostsdownstatus | stats max(livehostsdownstatus) AS livehostsdownstatus | gauge livehostsdownstatus [|inputlookup alldevices.csv | fields device | stats count(device) as max | eval first=0 | eval second=max*.50 | eval third=max*.75 | eval fourth=max  | eval range=first+" "+second+" "+third+" "+fourth | return $range ]

Nagios Hosts Down
index=nagios nagiosevent="HOST NOTIFICATION" hostnotificationstatus="DOWN"  | dedup hostnotificationstatus hostnotification | stats dc(src_host) as src_host | return src_host


Alerts Nagios
index=nagios (nagiosevent="SERVICE NOTIFICATION" severity="WARNING" OR severity="CRITICAL") OR (nagiosevent="HOST NOTIFICATION" hostnotificationstatus="DOWN") ( user_id="snmpoll*" OR user_id="nagiosadmin") | convert ctime(_time) as time  |transaction time,src_host | table time,src_host,user_id,reason,nagiosevent


index=nagios (nagiosevent="SERVICE NOTIFICATION" severity="WARNING" OR severity="CRITICAL") OR (nagiosevent="HOST NOTIFICATION" hostnotificationstatus="DOWN") ( user_id="snmpoll*" OR user_id="nagiosadmin") | convert ctime(_time) as time  |transaction src_host maxspan=20m | table time,src_host,user_id,reason,nagiosevent

Systems Events:
index=nagios (nagiosevent="SERVICE NOTIFICATION" ) OR (nagiosevent="HOST NOTIFICATION" ) ( user_id="ns_system*") | convert ctime(_time) as time | eval Name=coalesce(name,hostnotification) |transaction src_host,Name | table time,src_host,user_id,Name,reason,nagiosevent

index=nagios (nagiosevent="SERVICE NOTIFICATION" ) OR (nagiosevent="HOST NOTIFICATION" ) ( user_id="ns_system*")|lookup local=t nagios-hostgroupmembers host_name AS src_host | convert ctime(_time) as time | eval Name=coalesce(name,hostnotification) |transaction src_host,nagiosevent | table time,eventcount,src_host,hostgroup,user_id,Name,reason,nagiosevent 


Create Nagios Server lookup table


earliest=-24h index="nagios" nagiosevent="CURRENT HOST STATE"  | rex ".+CURRENT HOST STATE\: (?P<device>[^;]*)(?=;)"  | lookup local=true nagios-hostgroupmembers host_name AS src_host | search hostgroup=*server* | stats count by device | outputlookup server.csv


Create Nagios host group table
index=nagios host=nagios1-60ox.noc.harvard.edu sourcetype="nagios" nagiosevent="CURRENT HOST STATE" | fields src_host | lookup local=true nagios-hosts name AS src_host OUTPUT address AS src_ip, alias AS desc, hard_state | lookup local=true nagios-hostgroupmembers host_name AS src_host, OUTPUT state AS hard_state, num AS hard_num, hostgroup | outputlookup hostgroups.csv


Create nagios Service Groups table - not working right
index=nagios host=nagios1-60ox.noc.harvard.edu sourcetype="nagios" nagiosevent="CURRENT SERVICE STATE" |fields src_host, name| lookup local=true nagios-servicegroupmembers host_name AS src_host name, OUTPUT state AS hard_state, num AS hard_num, servicegroup | output lookup servicegroups.csv


index=nagios host=nagios1-60ox.noc.harvard.edu sourcetype="nagios" nagiosevent="CURRENT SERVICE STATE" | lookup local=true nagios-servicegroupmembers host_name AS src_host name, OUTPUT state AS hard_state, num AS hard_num, servicegroup | fields src_host, servicegroup | stats delim=\" values(servicegroup) as servicegroup by src_host | outputlookup servicegroups.csv

