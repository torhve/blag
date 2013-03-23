# Reusing munin plugins to get metrics into graphite

This is sort of part #3 of my previous blog articles [vm monitoring using salt and cubism](/vm-monitoring-using-salt-and-cubism) and [salt-returner-for-carbon](/salt-returner-for-carbon).
The new cog in the machine is getting metrics from [Munin](http://munin-monitoring.org/http://munin-monitoring.org/) since I already use Munin alot in my systems.

## Munin strengths and weaknesses

Munin is pretty awesome, and has a ton of plugins for getting metrics from different stuff (for example checkout their contrib tree: <https://github.com/munin-monitoring/contrib/tree/master/plugins>. It is also very easy to write new munin plugins.
The usual way to run munin is to have a daemon on every node, and then the master will then poke each minion every fifth minute to refresh data and put it into its RRD-files.

By using the combination of [Salt](http://saltstack.org/) and [Graphite](http://graphite.wikidot.com/) we do not have to use the munin node daemon, and instead let salt push metrics to Graphite much more often. There's tons of other projects that also interfaces nicely with graphite, and there is also other projects that helps with integrating munin with graphite, but since I already have salt-minion and munin on all my nodes, why not reuse existing component.

## Salt Munin module

I wrote a simple module for salt that uses the munin component *munin-run* to execute a plugin and parse the output into data that salt understands. The module has been accepted into Salt upstream. The critical function looks like this:

> salt/modules/munin.py

        muninout =  __salt__['cmd.run']('munin-run ' + plugin)
        data = {
            plugin: {}
        }
        for line in muninout.split('\n'):
            if 'value' in line: # This skips multigraph lines, etc
                key, val = line.split(' ')
                key = key.split('.')[0]
                try:
                    # We only want numbers
                    val = float(val)
                    data[plugin][key] = val
                except ValueError:
                    pass
        return data

You can easily run this on the CLI if you want to get metrics.
For example if you want to get the CPU stats from the CPU plugin:

    -> # salt salt\* munin.run cpu
    salt.demo.no:
        ----------
        cpu:
            ----------
            idle:
                559160602.0
            iowait:
                1104395.0
            irq:
                73.0
            nice:
                140745.0
            softirq:
                491340.0
            steal:
                801498.0
            system:
                39399629.0
            user:
                27544876.0


### Scheduling the munin to carbon pushing

I am reusing the carbon returner as described in the previous blog article in the series, and using the salt minion scheudler to run and push the data. The following salt config will run the munin module every tenth second to push data to carbon/graphite.

> /etc/salt/minion

    schedule:
      munin:
        function: munin.run
        args:
          - cpu
        seconds: 10
        returner: carbon.returner

As graphite users knows, there's many ways to display data, but this is an example of the result from the previous configuration using the default graphite browser.

![Muningraphite](http://hveem.no/ss/salt-munin-graphite.png)

The next logical step is combining this with Salt UI to dynamically define metrics you want to monitor into a dash board, so that you can easily define a screen that shows just the metric you want for solving a problem, or resolving a crisis etc. But that will be a blog article for another day.
