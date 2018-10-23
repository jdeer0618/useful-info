| tstats summariesonly=t latest(_time) AS _time FROM datamodel=Network_Traffic WHERE sourcetype=aws:cloudwatchlogs:vpcflow  BY  All_Traffic.src_ip All_Traffic.dest_ip
| eval pairKey = md5('All_Traffic.src_ip' . 'All_Traffic.dest_ip')
| eval data = "now"
| append 
    [| loadjob 1539812310.4128
     | eval pairKey = md5('All_Traffic.src_ip' . 'All_Traffic.dest_ip')]
| eventstats count AS notSeenBefore by pairKey
| where notSeenBefore = 1
| eval priority = if(cidrmatch("10.0.0.0/8", 'All_Traffic.dest_ip') OR cidrmatch("172.16.0.0/12", 'All_Traffic.dest_ip') OR cidrmatch("192.168.0.0/16", 'All_Traffic.dest_ip'), "normal", "elevated")
| stats count by priority data


| eval comment = "#######vpc flow"
| tstats summariesonly=t latest(_time) AS _time FROM datamodel=Network_Traffic WHERE sourcetype=aws:cloudwatchlogs:vpcflow earliest=-10d@d latest=@d BY  All_Traffic.src_ip All_Traffic.dest_ip
| `drop_dm_object_name("All_Traffic")`
| eval data = "historical" 
| eval src_ip_rfc_1918 = if(cidrmatch("10.0.0.0/8", 'src_ip') OR cidrmatch("172.16.0.0/12", 'src_ip') OR cidrmatch("192.168.0.0/16", 'src_ip'), "true", "false")
| eval dest_ip_rfc_1918 = if(cidrmatch("10.0.0.0/8", 'dest_ip') OR cidrmatch("172.16.0.0/12", 'dest_ip') OR cidrmatch("192.168.0.0/16", 'dest_ip'), "true", "false")
| where (src_ip_rfc_1918="true" AND dest_ip_rfc_1918="false") OR (src_ip_rfc_1918="true" AND dest_ip_rfc_1918="true")
| eval flow_dir = if((src_ip_rfc_1918="true" AND dest_ip_rfc_1918="false") OR (src_ip_rfc_1918="false" AND dest_ip_rfc_1918="true"), "vpc2public", "vpc2vpc")
| eval mvpair=md5(mvjoin(mvsort(mvappend(src_ip, dest_ip)), ""))
| outputlookup vpcflow_tracker.csv


| eval comment = "########more vpc flow"
| tstats summariesonly=t latest(_time) AS _time FROM datamodel=Network_Traffic WHERE sourcetype=aws:cloudwatchlogs:vpcflow earliest=-6d@d latest=@d BY  All_Traffic.src_ip All_Traffic.dest_ip
| `drop_dm_object_name("All_Traffic")`
| eval data = "historical" 
| eval pairKey = md5('src_ip' . 'dest_ip')
| eval rfc_dest = if(cidrmatch("10.0.0.0/8", 'dest_ip') OR cidrmatch("172.16.0.0/12", 'dest_ip') OR cidrmatch("192.168.0.0/16", 'dest_ip'), "true", "false")
| eval rfc_src = if(cidrmatch("10.0.0.0/8", 'src_ip') OR cidrmatch("172.16.0.0/12", 'src_ip') OR cidrmatch("192.168.0.0/16", 'src_ip'), "true", "false")
| where (rfc_src="true" AND rfc_dest="false") OR (rfc_src="true" AND rfc_dest="true")
| eval flow_dir = if((rfc_src="true" AND rfc_dest="false") OR (rfc_src="false" AND rfc_dest="true"), "vpc2public", "vpc2vpc")
| eval mvpair=mvappend(src_ip, dest_ip)
| eval mvpair = mvsort(mvpair)
| eval joined = mvjoin(mvpair, "")
| eval newMd5 = md5(joined)
| append 
    [| tstats summariesonly=t latest(_time) AS _time FROM datamodel=Network_Traffic WHERE sourcetype=aws:cloudwatchlogs:vpcflow earliest=@d latest=now BY  All_Traffic.src_ip All_Traffic.dest_ip
| `drop_dm_object_name("All_Traffic")`
| eval data = "now" 
| eval pairKey = md5('src_ip' . 'dest_ip')
| eval rfc_dest = if(cidrmatch("10.0.0.0/8", 'dest_ip') OR cidrmatch("172.16.0.0/12", 'dest_ip') OR cidrmatch("192.168.0.0/16", 'dest_ip'), "true", "false")
| eval rfc_src = if(cidrmatch("10.0.0.0/8", 'src_ip') OR cidrmatch("172.16.0.0/12", 'src_ip') OR cidrmatch("192.168.0.0/16", 'src_ip'), "true", "false")
| where (rfc_src="true" AND rfc_dest="false") OR (rfc_src="true" AND rfc_dest="true")
| eval flow_dir = if((rfc_src="true" AND rfc_dest="false") OR (rfc_src="false" AND rfc_dest="true"), "vpc2public", "vpc2vpc")
| eval mvpair=mvappend(src_ip, dest_ip)
| eval mvpair = mvsort(mvpair)
| eval joined = mvjoin(mvpair, "")
| eval newMd5 = md5(joined)]
| where flow_dir = "vpc2vpc"
| eventstats count by newMd5
| where data="now" AND count=1


| eval comment = "#####################more cowbell"
| tstats summariesonly=t latest(_time) AS _time FROM datamodel=Network_Traffic WHERE sourcetype=aws:cloudwatchlogs:vpcflow earliest=@d latest=@d+5h BY  All_Traffic.src_ip All_Traffic.dest_ip
| `drop_dm_object_name("All_Traffic")`
| eval data = "historical" 
| eval src_ip_rfc_1918 = if(cidrmatch("10.0.0.0/8", 'src_ip') OR cidrmatch("172.16.0.0/12", 'src_ip') OR cidrmatch("192.168.0.0/16", 'src_ip'), "true", "false")
| eval dest_ip_rfc_1918 = if(cidrmatch("10.0.0.0/8", 'dest_ip') OR cidrmatch("172.16.0.0/12", 'dest_ip') OR cidrmatch("192.168.0.0/16", 'dest_ip'), "true", "false")
| where (src_ip_rfc_1918="true" AND dest_ip_rfc_1918="false") OR (src_ip_rfc_1918="true" AND dest_ip_rfc_1918="true")
| eval flow_dir = if((src_ip_rfc_1918="true" AND dest_ip_rfc_1918="false") OR (src_ip_rfc_1918="false" AND dest_ip_rfc_1918="true"), "vpc2public", "vpc2vpc")
| eval mvpair=md5(mvjoin(mvsort(mvappend(src_ip, dest_ip)), ""))
| inputlookup append=t vpcflow_tracker.csv
| eventstats max(_time) AS time by mvpair
| where time > _time


| eval comment = "##################random timestamp"
| makeresults count=100
| eval random = random() %10000
| eval now = now()
| eval len = len(random)
| eval timeLen = len(now) - 'len'
| eval n = substr('now', 1, timeLen)
| eval _time = n . random


| eval comment = "###################latest vpcflow"
| tstats summariesonly=t latest(_time) AS _time FROM datamodel=Network_Traffic WHERE sourcetype=aws:cloudwatchlogs:vpcflow earliest=-1d@d latest=@d BY All_Traffic.src_ip All_Traffic.dest_ip 
| `drop_dm_object_name("All_Traffic")` 
| eval mvpair=md5(mvjoin(mvsort(mvappend(src_ip, dest_ip)), "")) 
| sort 0 -_time 
| dedup mvpair 
| eval src_ip_rfc_1918 = if(cidrmatch("10.0.0.0/8", 'src_ip') OR cidrmatch("172.16.0.0/12", 'src_ip') OR cidrmatch("192.168.0.0/16", 'src_ip'), "true", "false") 
| eval dest_ip_rfc_1918 = if(cidrmatch("10.0.0.0/8", 'dest_ip') OR cidrmatch("172.16.0.0/12", 'dest_ip') OR cidrmatch("192.168.0.0/16", 'dest_ip'), "true", "false") 
| where (src_ip_rfc_1918="true" AND dest_ip_rfc_1918="false") OR (src_ip_rfc_1918="true" AND dest_ip_rfc_1918="true") 
| eval flow_dir = if((src_ip_rfc_1918="true" AND dest_ip_rfc_1918="false") OR (src_ip_rfc_1918="false" AND dest_ip_rfc_1918="true"), "vpc2public", "vpc2vpc") 
| eval data = "new" 
| appendpipe 
    [| inputlookup vpcflow_tracker.csv 
    | eval data = "old"] 
| where (now() - _time) / 86400 < 30 
| outputlookup vpcflow_tracker.csv 
| eventstats count(mvpair) AS pairCount by mvpair 
| where pairCount = 1 AND data = "new"


| tstats summariesonly=t latest(_time) AS _time FROM datamodel=Network_Traffic WHERE sourcetype=aws:cloudwatchlogs:vpcflow All_Traffic.src_ip!="-" earliest=@d latest=now BY All_Traffic.src_ip All_Traffic.dest_ip source
| `drop_dm_object_name("All_Traffic")`
| rex field=source "^(?<region>[^\:]+)\:(?<vpc_group>[^\:]+)\:(?<interface_id>.+)-all"
