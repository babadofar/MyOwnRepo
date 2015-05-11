# Analyzing weblogs with Elasticsearch in the cloud: Using Logstash & Kibana on Found by Elastic pt. II
 
This is part two in the series using Elasticsearch, Kibana and Logstash with Found by Elastic. This time we will use Logstash to feed logs from a web search application running on the Microsoft web server IIS (Internet Information Service)  into Elasticsearch, and use Kibana to show some nice graphs. 

## Logstash
Logstash is a tool for pipeline processing of logs. It can read logs from many different sources, has a flexible framework for processing them, and a large array of output formats. In this post we will read input from a file, split the raw text line into fields, and process it in various ways. Finally the set of nicely formatted fields is sent into Elasticsearch. Outputting to Elasticsearch allows us to use the powerhorse of visualizations, Kibana 4. 

Logstash comes with an extensive set of [pre built log patterns](https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns), but, unfortunately, IIS, unlike Apache, doesn’t have a standard log format, so we need to create our own [grok pattern](http://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html#_grok_basics) to parse it. 

````
 grok {
    # TODO: check that this pattern matches your IIS log settings
    match => ["message", "%{TIMESTAMP_ISO8601:log_timestamp} %{IPORHOST:site} %{NOTSPACE:method} %{URIPATH:page} (?:%{NOTSPACE:querystring}|-) %{NUMBER:port} (%{WORD:username}|-) %{IPORHOST:clienthost} %{NOTSPACE:useragent} %{NOTSPACE:referer} %{NUMBER:response} %{NUMBER:subresponse} %{NUMBER:scstatus} %{NUMBER:time_taken}"]
  }
 
  #Set the Event Timestamp from the log
  date {
    match => [ "log_timestamp", "YYYY-MM-dd HH:mm:ss" ]
    timezone => "Etc/UCT"
  }
  ````

The full config file for IIS can be found at [gist](https://gist.github.com/babadofar/5fbea416c6a07ca209bf)

Again, there is no standard defined format for IIS logs, so while this grokked my logs, it might not grok yours.  The IIS logs are from a search application, and contain the search terms in a query string parameter called, of course, query. 

## Grok development
One of the most convenient ways to work with creating grok patterns is [the Grok Constructor](http://grokconstructor.appspot.com/do/construction)   Paste in some sample log lines, and press Go!. A long list of suggested matches will be displayed at the bottom screen. Select one that most closely matches your data - for IIS logs, the first field should be some kind of date. 



## Running Logstash
Download the gist file, and modify the file location to match your logs. Wildcards should work fine.  Run it with this command: 

````
logstash.bat agent --verbose -f iisClient.conf
````

Logstash is normally running very quietly, setting the --verbose flag outputs some more information. The configuration is also outputting debug information for each log line.


## Kibana
Start up Kibana while Logstash is running, to see the progress as it works.
Kibana opens up to a screen where you have to select an index pattern, and a timestamp. For this project you can select the default suggested index pattern, and you should also use the default @timestamp field for the Time-field name.
If you don’t see any data, either nothing reached Elasticsearch, or you need to expand the date range from the datepicker in the upper right corner.   

Clicking on the [Discover](http://www.elastic.co/guide/en/kibana/current/discover.html) tab shows a list of search results, the number of hits on the right side, and on the left a list of fields from your documents. Clicking on a field reveals some quick statistics on the contents, gathered from a random sample of 500 documents

## Graphing Most used query terms in Kibana
For a search application, the most interesting metrics is perhaps the most used query terms. 
Clicking on the "Visualize" button at the bottom of the field metrics opens up the Visualize tab showing a visualization of most used query terms. If you want the analysis to show complete queries, and not broken up into terms, use the "query.raw" field rather than "query". Logstash by default indexes all fields into two fields, one analyzed, where all terms are split, and the other not analyzed. The not analyzed fields are useful when you need statistics on the whole field value.


## Graphing Number of Requests
Enter the Visualize panel, and select to create a new visualization. Select "Line chart" as type of visualization, from a new search. Press the Add aggregation button, and select Aggregation type Date histogram, and the field @timestamp. Press Apply at the bottom. If you have data (in the applied date range), you should see a graph similar to this one. 



## Graphing Response Times
If you looked closely at the Logstash configuration, you will find that it  contains a section where a field “time_taken” is converted into integer. By default, Logstash will output all fields as strings, to be able to use the Histogram aggregation, we have to be explicit about making sure the field has the right type. 
Again, select to start a new visualization, of type “Line Chart”, from a new search. Now we are going to modify the Y-Axis, setting the type of Aggregation to Average, and using the field “time_taken” as input. For X-Axis we will use the Date histogram and the @timestamp field. Now you should see a graph displaying how the average response times varies over time.



This is just a scratch on the surface of how to work with visualizing log statistics using the ELK stack. 
