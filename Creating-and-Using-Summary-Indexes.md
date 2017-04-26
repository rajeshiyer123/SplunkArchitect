# Creating and Using Summary Indexes


## Saved Search to Populate a Summary Index

You can create a Saved Search that runs periodically to populate the summary index with data for later retreival:

(Search Head)  
Settings > Searches, reports, and alerts  
Select the App context to create the saved search in, if applicable  
New  

Complete the fields per the following image and field data examples for a report named 'Top 20 NGE Exception Messages':  

![Saved Search Report Setup 1](/images/saved_search_settings_1.png)  

The report is configured to collect data from the previous whole hour using the Earliest = -1h@h & Latest = -0h@h time specifiers.  

The search string used in the above report is:  
```
eventtype=fpp_gxp_services sourcetype=disney_nge_xbms nge_exception_message=* | rex field=nge_exception_message mode=sed "s/(#\d+])/#xxxx]/g" 
| rex field=nge_exception_message mode=sed "s/(\[.+])/[xxxx]/g"  
| rex field=nge_exception_message mode=sed "s/(Transaction timed out: deadline was .+$)/Transaction timed out: deadline was -date-/g"
| rex field=nge_exception_message mode=sed "s/(guest\/.*\/identifiers)/guest\/xxxx\/identifiers/g" 
| rex field=nge_exception_message mode=sed "s/(ID: .*$)/ID: xxxx/g" 
| rex field=nge_exception_message mode=sed "s/(For input string: ".*")/For input string "xxxx"/g" 
| rex field=nge_exception_message mode=sed "s/(End Date .* is prior to .*$)/End Date -date- is prior to -date-/g" 
| rex field=nge_exception_message mode=sed "s/(party for \d+ entitlement$)/party for xxxx entitlement/g" 
| rex field=nge_exception_message mode=sed "s/(found for \d+ for)/found for xxxx for/g" 
| top 20 nge_exception_message useother=true
| addinfo 
| eval start=strftime(info_min_time, "%Y-%m-%d %T") 
| eval end=strftime(info_max_time, "%Y-%m-%d %T") 
| eval _time = info_max_time
| fields start end count percent nge_exception_message 
```

__Notes on Search String Attributes__

* The 'rex' entries remove and replace identifiers that may be unique with generic 'xxxx' or '-data-' values so that the exception messages are reduced to just their 'types'.

* addinfo adds time info fields to each event including:  

	* info_min_time  
	* info_max_time  
	* info_search_time  

* The __eval _time = info_max_time__ creates a timestamp for the event from the info_max_time epoch value that reflects the *end* of the time range. The _time timestamp would otherwise reflect the time of the *start* of the time range, which can be misleading and doesn't lend itself well to time series analysis.  

* The designated fields, along with the rest of the addinfo fields, are added to the designated summary index as events.


![Saved Search Report Setup 2](/images/saved_search_settings_2.png)  

![Saved Search Report Setup 3](/images/saved_search_settings_3.png)  



Saving the scheduled report with summary indexing creates the following entries in ```/opt/splunk/etc/apps/wdpr_bog_dashboard/local/savedsearches.conf```; __notice the additional 'datasource' field__ which has been added so these events can be found / isolated from a summary index that may contain other event types:

```
[Top 20 NGE Exception Messages]
action.summary_index = 1
action.summary_index._name = summary_nge_exception_messages
action.summary_index.datasource = top_20_nge_exception_messages
alert.digest_mode = True
alert.suppress = 0
alert.track = 0
auto_summarize.dispatch.earliest_time = -1d@h
cron_schedule = 5 * * * *
description = Top 20 NGE Exception Messages by hour
dispatch.earliest_time = -1h@h
dispatch.latest_time = -0h@h
enableSched = 1
realtime_schedule = 0
search = eventtype=fpp_gxp_services sourcetype=disney_nge_xbms nge_exception_message=* earliest=-1h@h latest=-0h@h \
| rex field=nge_exception_message mode=sed "s/(#\d+])/#xxxx]/g" \
| rex field=nge_exception_message mode=sed "s/(\[.+])/[xxxx]/g"  \
| rex field=nge_exception_message mode=sed "s/(Transaction timed out: deadline was .+$)/Transaction timed out: deadline was -date-/g"\
| rex field=nge_exception_message mode=sed "s/(guest\/.*\/identifiers)/guest\/xxxx\/identifiers/g" \
| rex field=nge_exception_message mode=sed "s/(ID: .*$)/ID: xxxx/g" \
| rex field=nge_exception_message mode=sed "s/(For input string: ".*")/For input string "xxxx"/g" \
| rex field=nge_exception_message mode=sed "s/(End Date .* is prior to .*$)/End Date -date- is prior to -date-/g" \
| rex field=nge_exception_message mode=sed "s/(party for \d+ entitlement$)/party for xxxx entitlement/g" \
| rex field=nge_exception_message mode=sed "s/(found for \d+ for)/found for xxxx for/g" \
| top 20 nge_exception_message useother=true\
| addinfo \
| eval start=strftime(info_min_time, "%Y-%m-%d %T") \
| eval end=strftime(info_max_time, "%Y-%m-%d %T") \
| fields start end count percent nge_exception_message\
```



> Written with [StackEdit](https://stackedit.io/).