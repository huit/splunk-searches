Time-Correlated Errors search correlating Nagios High System Load for network devices against errors in the cisco index

(index=nagios perfdata=SERVICEPERFDATA name="System Load" severity="WARNING" OR secerity="CRITICAL" )  OR (index=cisco tag=error NOT host=*wlc* NOT host=*wrls* )| rex field=load_1_min "(?<cpu_load>.+)%" | eval epoch=round(_time/ (60 * 5), 0) | eval correlated_issues=if(sourcetype == "nagiosserviceperf", null, sourcetype + " | " + _raw) | eval error_time=if(sourcetype == "nagiosserviceperf", strftime(_time, "%m-%d-%y %H: %M"), null) | stats list(error_time) AS error_time list(src_host) AS src_host list(correlated_issues) AS correlated_issues BY epoch | search error_time=* src_host=* correlated_issues=*  | sort - epoch | fields - epoch

I think this is actually a better version

( index=nagios statusnotification=CRITICAL OR statusnotification=WARNING )  OR (index=cisco tag=error NOT host=*wlc* NOT host=*wrls* ) OR ( index=fwsm tag=error)  | eval epoch=round(_time/ (60 * 5), 0) | eval correlated_issues=if(sourcetype == "nagios", null, sourcetype + " | " + _raw) | eval error_time=if(sourcetype == "nagios", strftime(_time, "%m-%d-%y %H: %M"), null) | stats list(error_time) AS error_time list(src_host) AS src_host list(correlated_issues) AS correlated_issues BY epoch | search error_time=* src_host=* correlated_issues=*  | sort - epoch | fields - epoch

Compare volume 3 weeks vs this week - showing anomolies with std dev, anything outside of 3 is troublsome

index=cisco tag=error NOT host=*wlc* earliest=-1mon@d latest=@h | bucket _time span=1h | eval day=if(_time >= relative_time(now(), "-7d@d"), "this", "past") | stats count AS c BY _time date_wday date_mday date_hour sourcetype day | eval c_tmp=c | eval c=if(day == "this", null, c) | eventstats mean(c) AS m stdevp(c) AS sd BY date_wday date_hour sourcetype | rename c_tmp AS c | where (day = "this") AND (date_mday = tonumber(strftime(now(), "%d"))) | eval z=(c - m)/sd | xyseries _time sourcetype z | eval Hour=strftime(_time, "%H") | fields - _time


Relative Ratios to compare today VS yesterday, above 0 is addiional value today and below 0 the magnatude of yesterdays volume

by sourcetype: 

index=cisco tag=error NOT host=*wlc* earliest=-1d@d latest=@h | where date_hour < tonumber(strftime(now(), "%H")) | eval day=if(_time < relative_time(now(), "@d"), "yesterday", "today") | stats count AS c BY date_hour sourcetype day | eval yesterday=if(day == "yesterday", c, 0) | eval today=if(day == "today", c, 0) | stats max(yesterday) AS yesterday max(today) AS today BY date_hour sourcetype | eval ratio=if(today >= yesterday, today/ yesterday, -yesterday/today) | fields - yesterday today | rename date_hour AS Hour | xyseries Hour sourcetype ratio | sort + Hour

by host:

index=cisco tag=error NOT host=*wlc* earliest=-1d@d latest=@h | where date_hour < tonumber(strftime(now(), "%H")) | eval day=if(_time < relative_time(now(), "@d"), "yesterday", "today") | stats count AS c BY date_hour host day | eval yesterday=if(day == "yesterday", c, 0) | eval today=if(day == "today", c, 0) | stats max(yesterday) AS yesterday max(today) AS today BY date_hour host | eval ratio=if(today >= yesterday, today/ yesterday, -yesterday/today) | fields - yesterday today | rename date_hour AS Hour | xyseries Hour host ratio | sort + Hour

Quick and dirty discovery of rare events
index=cisco tag=error NOT host=*wlc* | cluster t=.8 field=_raw match=termset | table _raw cluster_count | sort + cluster_count
