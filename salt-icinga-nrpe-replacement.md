# How I taught Icinga how to use Salt Stack for asynchrynous distributed passive checks
 **a NRPE replacement**

## Overview## 

The usual way for Icinga/Nagios to run checks that run on the target OS it's checking is to use the NRPE agent daemon installed on the target OS, and then icinga forks all day long to run check_nrpe and communicating directly with the remote agent. This works, but I never liked it. It's inefficient/slow and doesn't scale very well.
Thus, my plan is to utilize [the Salt Stack](http://saltstack.org) for asynchronous distributed icinga checks.
Salt uses the [PUB/SUB pattern](http://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) and its usual mode of operation is run commands on the master and have minions return their answers. It has a couple of other modes too, and I will be using the [Peer Runner](http://docs.saltstack.org/en/latest/ref/peer.html). The Peer Communication capability of Salt lets you publish a command from one peer to be executed on other peers.

## The NRPE Salt Module ## 
A salt module is defined in the [salt documentation]("http://docs.saltstack.org/en/latest/index.html#remote-execution) as:

> Salt modules are the core of remote execution. They provide functionality such as installing a package, restarting a service, running a remote command, transferring a file — and the list goes on.

The first component I needed for this setup is a NRPE module to run existing NRPE checks defined in the NRPE configuration file.

## /srv/salt/base/_modules/nrpe.py ## 

    #!/usr/bin/env python
    '''
    Support for running nrpe over salt instead of nrpe daemon
    '''

    # Import python libs
    import os
    import argparse
    import re

    # Import salt libs
    import salt.utils
    from salt.exceptions import SaltException

    def __virtual__():
        '''
        Only load the module if nrpe is installed
        '''
        if os.path.exists('/etc/nagios/nrpe.cfg'):
            return 'nrpe'
        return False

    def run(name):
        '''
        Look in /etc/nagios/nrpe.cfg for name and run the command configured. Return the retcode and output   '''

        fp = salt.utils.fopen('/etc/nagios/nrpe.cfg', 'r')
        cmd = re.search(r'command\[%s]=(?P<command>.*)' %name, fp.read())
        if cmd:
            command = cmd.groupdict()['command']
            return __salt__['cmd.run_all'](command)
        else:
            return 'Command with name "%s" not found' %name

## The NRPE Salt Runner## 

A salt runner is defined by the [the salt documentation site](https://salt.readthedocs.org/en/latest/ref/runners/index.html?highlight=runner) as:
> Salt runners are convenience applications executed with the salt-run command.
The use for a Salt runner is to build a frontend hook for running sets of commands via Salt or creating special formatted output.

I wrote a simple runner that takes the check to be ran as argument. This can later be extended to run on certain hostgroups, or run named checks with argument or do more/better formatting and security checks.

## runners/nrpe.py## 

    '''
    Run a icinga/nagios NRPE check using the nrpe module and return the proper exit code and output
    '''

    # Import salt libs
    import salt.client
    import sys


    def check(name):
        '''
        Run a named check
        '''
        client = salt.client.LocalClient(__opts__['conf_file'])
        ret = client.cmd('*', 'nrpe.run', arg=(name,), timeout=__opts__['timeout'])
        salt.output.display_output(ret, 'yaml', __opts__)
        return ret

## Salt peer configuration## 
We have to tell salt master that our icinga server can publish (run) the NRPE runner:
## /etc/salt/master## 
    peer_run:
    icinga:
    - nrpe.* 

Alright! Let's see if the runner is working by doing a run of the check_load NRPE check:

    root@icinga # salt-call --out=json publish.runner nrpe.check check_load
    "local": {
        "icinga": {
            "pid": 7035,
            "retcode": 0,
            "stderr": "",
            "stdout": "OK - load average: 2.40, 1.41, 0.89|load1=2.400;20.000;30.000;0; load5=1.410;15.000;25.000;0; load15=0.890;10.000;20.000;0;"
        "salt": {
            "pid": 23298,
            "retcode": 0,
            "stderr": "",
                "stdout": "OK - load average: 0.01, 0.04, 0.05|load1=0.010;15.000;30.000;0; load5=0.040;10.000;25.000;0; load15=0.050;5.000;20.000;0;"
            }
        }
    }

Looks like the load is bearable on the icinga and salt servers.

## The Icinga feeder## 

The last piece of the puzzle is a program that will read the output from the runner and convert this into icinga external process check format like specified in the [icinga passive checks](http://docs.icinga.org/latest/en/passivechecks.html) interface. 
It should be run at specified intervals, publish the runner with the named check, parse the result using one of the salt output formats and finally write the results into the external command file.
If one configures the freshness settings correctly, what happens if there's problem with salt checks, then icinga will fall back to running active checks using the NRPE daemon again like before.

## salt2icinga.py## 

    #!/usr/bin/env python
    '''
    Simple script to parse output from salt-call runnner and feed into icinga
    (C) 2013 Tor Hveem <tor@hveem.no>
    '''
    import yaml
    import sys
    import optparse
    import tempfile
    from time import time

    def write_external_cmd(cmd_file, status_file):
      ''' Submits the status lines in the status_file to Nagios' external cmd file.
      '''
      try:
        with open(cmd_file, 'a') as cmd_file:
          cmd_file.write(status_file.read())
      except IOError:
        exit("Fatal error: Unable to write to Icinga external command file '%s'.\n"
             "Make sure that the file exists and is writable." % (cmd_file,))


    if __name__ == '__main__':
        # optionparser for command file
        parser = optparse.OptionParser()
        parser.add_option("-c", "--cmd-file", metavar="FILE",
                             help="Path to the file that Nagios checks for "
                               "external command requests.")
        parser.add_option("-n", "--name", metavar="NAME",
                             help="Named check to run")

        (options, args) = parser.parse_args()

        data = yaml.load(sys.stdin.read())
        if not data:
            print "No data from stdin to read"
            sys.exit(1)

        tmp_status_file = tempfile.TemporaryFile()
        CMD_STATUS_FORMAT = "[{0}] PROCESS_SERVICE_CHECK_RESULT;{1};{2};{3};{4}\n"
        timestamp = long(time()) # approximate time the services were queried
        for host, ret in data['local'].items(): # local since we are using salt-call
            if type(ret) != type({}): # Hosts without check defined will return string result
                continue
            # [<timestamp>] PROCESS_SERVICE_CHECK_RESULT;<host>;<service>;<return_code>;<message>\n
            status_line = CMD_STATUS_FORMAT.format(
                timestamp, host, options.name, ret['retcode'], ret['stdout'])
            tmp_status_file.write(status_line)
        tmp_status_file.flush()
        tmp_status_file.seek(0)
        write_external_cmd(options.cmd_file, tmp_status_file)

Lets see if it works:
    root@icinga # salt-call -l quiet --out=yaml publish.runner nrpe.check check_users | python salt2icinga.py -n check_load -c /dev/stdout

    [1357425575] PROCESS_SERVICE_CHECK_RESULT;icinga;check_load;0;OK - load average: 2.24, 1.92, 1.38|load1=2.240;20.000;30.000;0; load5=1.920;15.000;25.000;0; load15=1.380;10.000;20.000;0;
    [1357425575] PROCESS_SERVICE_CHECK_RESULT;salt;check_load;0;OK - load average: 0.00, 0.01, 0.05|load1=0.000;15.000;30.000;0; load5=0.010;10.000;25.000;0; load15=0.050;5.000;20.000;0;

**Huzzah!** Mission accomplished.

All that's needed now is a salt state file to write all the cronjobs to schedule the checks and write them to the icinga external cmd file

Check out [Part #2](/icinga-configuration-generation-using-salt) about configuration generation.

Thanks for reading. Contact me if you have any suggestions for improvements, etc.

