# debugging_splunk_search
This process outlines the steps to collect eu-stacks on a splunk search process pid.

Typically this level of debugging is performed to further debug a slow search and can be requested from Splunk Support. 

install the collect-pstack.tar.gz on the SH the search will be run from as well as the slowest indexer. 
You can check how long each indexer took on the search by looking in the Job Inspector after the search finishes:>Job > Inspect Job

Execution costs:
0.53 	dispatch.stream.remote 	6 	- 	977,934
Duration bar image 	0.30 	dispatch.stream.remote.rplinux-IDX2 	3 	- 	465,685
Duration bar image 	0.13 	dispatch.stream.remote.rplinux-IDX1 	1 	- 	55,484
Duration bar image 	0.10 	dispatch.stream.remote.rplinux-IDX3 	2 	- 	456,765 

In the above output of a job inspector we can see that there were three indexers which returned results where rplinux-IDX2 took the longest (.30 sec) and sent 465,685 bytes to the search head.



•	Install eu-stack:

sudo apt install elfutils

or 
yum install elfutils


•	Download the package from here:
https://github.com/rephillips/debugging_splunk_search/blob/main/collect-pstack.tar.gz

•	Put it under $SPLUNK_HOME/bin/, untar it there, there should be collect.sh scripts:

[bin]$ ls collect* collect-command.sh collect-data-all.py collect-pstack.tar.gz collect.sh collect-stacks.sh
•	Start the script by simply "./collect.sh" and keep it running:


turn on debug_metrics on the SH and the slowest indexer you will collect from 

/opt/splunk/etc/system/local/limits.conf
[search_metrics]
debug_metrics = true
#no splunk restart required 


[bin]$ ./collect.sh 
Setup Splunk Env Tab-completion of "splunk <verb> <object>" is available. Start pstack on splunkd ...
•	Run the test search with debug and "PSTACKME": 

  ie: test search or your slow search (try to keep search as simple as possible):
index=_internal sourcetype=splunkd earliest=11/05/2021:18:00:00 latest=11/05/2021:18:05:00 | noop log_appender="searchprocessAppender;maxFileSize=50000000;maxBackupIndex=99" log_debug=* set_ttl=30m | fields *,PSTACKME

- check the job inspector to get the sid of the search
- the stacks will be inside the search artifact directory of the sid of the search in : 
SH: $SPLUNK_HOME/var/run/splunk/dispatch/<sid>
Indexer: $SPLUNK_HOME/var/run/splunk/dispatch/remote_<SH>_<sid>

 !NOTE!: by default an ad-hoc search artifact will be deleted after 10m. You will want to move the artifact directory off to /tmp to preserve it.

•	Collect and send to Splunk Support:
SH:
a.)job inspector export (pdf)
b.) search artifact of desired sid (confirm stacks are within )
c.) diag

Indexer(s):
a.) search artifact of desired sid (confirm stacks are within )
b.) diag

to collect a diag: 
$SPLUNK_HOME/bin
  ./splunk diag
  
  
  turn off debug_metrics:
  
  /opt/splunk/etc/system/local/limits.conf
[search_metrics]
debug_metrics = false
#no splunk restart required 

a note on the log_debug: 
| noop log_appender="searchprocessAppender;maxFileSize=50000000;maxBackupIndex=99" log_debug=* set_ttl=30m
  

This command will add debug to a single search process and increase the search artifact's search.log file size from default of 25MB per file to 50MB per file and number of search.log.x files from default of 3 to 99. and set all channels to DEBUG which will increase logging verbosity for that search as well as extend the ttl of the artifact from default 10m of an ad-hoc search to 30m.
both SH and indexers will get the change.
  
  Add debugging to a search process:
https://docs.splunk.com/Documentation/Splunk/8.2.1/SearchReference/noop

For generating searches:

ie:

index=_internal sourcetype=splunkd | noop log_appender="searchprocessAppender;maxFileSize=50000000;maxBackupIndex=99" log_debug=* set_ttl=30m

For transforming searches: (such as stats , timechart ) add it before the first pipe:

index=_internal sourcetype=splunkd | noop log_appender="searchprocessAppender;maxFileSize=50000000;maxBackupIndex=99" log_debug=* set_ttl=30m | stats count by host

index=_internal sourcetype=splunkd | noop log_appender="searchprocessAppender;maxFileSize=50000000;maxBackupIndex=99" log_debug=* set_ttl=30m | timechart count by host


