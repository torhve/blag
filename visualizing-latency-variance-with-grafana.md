Visualizing latency variance with Grafana
=========================================

Measuring ICMP latency is a core tool for monitoring your network performance and is a common method for discovering symptoms that can reveal many different problems. [Smokeping](http://oss.oetiker.ch/smokeping/doc/reading.en.html) is best tool I know of that can display latency variance nicely. Smokeping uses [RRDtool](http://oss.oetiker.ch/rrdtool/) as its underlying database, but RRDtool has big problems with scaling if you have a huge number of metrics. There is a number of new contenders in the time series database space and I wanted to investigate and learn more about them. For this project I chose to use [Influxdb](http://influxdb.com/), which markets itself as a time series, metrics, and analytics database.

[Grafana](http://grafana.org) is a HTML5 dashboard tool which can gather metrics from different databases that enables you to select metrics visualize them.

So my goal is to see if I could come close to Smokeping's way of visualizing latency variance using "smoke" with Grafan's built in capabilities.

> Example of Smokeping's graph from [smokeping's site](http://oss.oetiker.ch/smokeping/doc/reading.en.html)

![Smokeping graph](//hveem.no/static/reading_detail.png)

In the spirit of learning I wrote a [small program](https://github.com/torhve/infping) in [Go](http://golang.org) that parses the output of [fping](http://fping.sourceforge.net/) and stores the results in Influxdb. Add hosts in the `configuration.toml` file and run it to start storing metrics in influxdb.

After I've gotten the metrics stored, I installed the development version of Grafana which has support for Influxdb 0.9. The whole trick to this thing is storing each output field from fping as separate series in influxdb. That means `min`, `avg`, `max`, and `loss`, and then selecting them as different series for grafana to graph. Let me show you:

![Grafana metrics](https://hveem.no/ss/2015-05-22_15-17-22.png)

And after giving aliases to the different series we choose some different display styles for each. I will use fill below to get it to draw the area between max and mean, and mean and min. The loss series will be drawn as a bar chart in red to really display where there's been actual loss of packages. Also the min for that series gets set to 0 so it won't clutter the graph when there's no loss. Again, let me show you with a screenshot of the settings:

![Grafana display styles](https://hveem.no/ss/2015-05-22_15-20-14.png)

End result
----------

The end result is in my opinion a really nice result which gives a lot of useful information in a single chart:

![Smokeping](https://hveem.no/ss/2015-05-22_15-21-52.png)

Zoomed in:

![Smokeping zoom](https://hveem.no/ss/2015-05-22_15-22-51.png)

Zoomed out to a week

![Smokeping week](https://hveem.no/ss/2015-05-22_15-24-49.png)

