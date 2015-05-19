# Analyzing weblogs with Elasticsearch in the cloud: Using Logstash & Kibana on Found by Elastic pt. II
 
This is part two in the series using Elasticsearch, Kibana and Logstash with Found by Elastic. In the previous post, we setup Logstash 1.5 and Kibana 4.02 to communicate with Elasticsearch in the cloud. This time we will use Logstash to feed logs from a web search application running on the Microsoft web server IIS (Internet Information Service)  into Elasticsearch, and use Kibana to show some nice graphs. 

## Logstash
Logstash is a tool for pipeline processing of logs. It can read logs from many different sources, has a flexible processing framework, and a wide selection of outputs. In this post we will read input from a file, split the raw text line into fields,  and process it to get the user query terms. Finally the set of nicely formatted fields is sent off into Elasticsearch. 

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
Download the gist file, and save it as iisClient.conf. Modify the file location to match the location of your logs. Wildcards should work fine (we're using Logstash 1.5 here).  Run it with this command: 

````
logstash.bat agent --verbose -f iisClient.conf
````

Logstash is normally running very quietly, we're using the --verbose flag to get a little more details on what it's doing. You can use --debug if you're really curious. The Logstash configuration is also outputting each event to the console, using the stdout output and rubydebug codec.


## Kibana
You can start up Kibana while Logstash is running, to see the progress as Logstash works.
On opening Kibana, you are welcomed to a screen that offers you to select an index pattern, and a timestamp. For this project, select the default suggested index pattern, and the default @timestamp field for the Time-field name.
If you don’t see any data, either nothing reached Elasticsearch, or you need to expand the date range from the datepicker in the upper right corner.   

Clicking on the [Discover](http://www.elastic.co/guide/en/kibana/current/discover.html) tab shows a list of search results, the number of hits on the right side, and on the left a list of fields from your documents. Clicking on a field reveals some quick statistics on the contents, gathered from a random sample of 500 documents

## Graphing Most used query terms in Kibana
For a search application, the most interesting metrics is perhaps the most used query terms. Select "query" from the list of fields on the left hand side. You will see something similar to this[!FieldStats] (https://raw.githubusercontent.com/babadofar/MyOwnRepo/master/images/KibanaDiscoverFieldStats.png)
Clicking on the "Visualize" button at the bottom of the field metrics opens up the Visualize tab with a bar chart of most used query terms. If you want the analysis to show complete queries, and not broken up into terms, select the "query.raw" field rather than "query". Logstash by default indexes all fields twice, one using the default analyzer, and the other not analyzed. For analyzed fields, Elasticsearch splits the field contents into individual terms (tokens), which is very useful when searching. However, when analyzing user behavior and dealing with statistics, you often need the complete query string given by the user, this will be available in the not analyzed version of the field, which you can access by using the "field.raw" notation. ![Term Raw Selector](https://raw.githubusercontent.com/babadofar/MyOwnRepo/master/images/KibanaTermRawSelector.png)

![Most used queries](https://raw.githubusercontent.com/babadofar/MyOwnRepo/master/images/mostUsedQueries.png)
The distribution of search queries normally follow something similar to Zipf's law, or the 80/20 rule, where 20% of the queries account for 80% of traffic. 


## Graphing Number of Requests
Enter the Visualize panel, and select to create a new visualization. Select "Line chart" as type of visualization, from a new search. Press the Add aggregation button, and select Aggregation type Date histogram, and the field @timestamp. Press Apply at the bottom. If you have data (in the applied date range), you should see a graph similar to this one. 
![Requests pr second](https://raw.githubusercontent.com/babadofar/MyOwnRepo/master/images/requestsPrSecond.png)


## Graphing Response Times
If you looked closely at the Logstash configuration, you will find that it contains a section where a field "time_taken" is converted into integer. By default, Logstash will output all fields as strings, to be able to use the Histogram aggregation, we have to be explicit about making sure the field has the right type. 
Again, select to start a new visualization, of type "Line Chart", from a new search. Now we are going to modify the Y-Axis, setting the type of Aggregation to Average, and using the field "time_taken" as input. For X-Axis we will use the Date histogram and the @timestamp field. Now you should see a graph displaying how the average response times varies over time.
![Average Response Time](https://raw.githubusercontent.com/babadofar/MyOwnRepo/master/images/AverageResponseTimes.png)


This is just a scratch on the surface of how to work with visualizing log statistics using the ELK stack. Hopefully, this will be enough to start exploring the ways of the ELK-stack.
