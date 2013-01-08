# Using salt to generate configuration for remote checks

Who likes writing icinga configuration ? Certainly not me. But I have a configuration management and remote execution system called *Salt* that can help me alleviate my pain. 
The two pronged attack consists of :
- a Salt module for NRPE, it reads the nrpe.cfg on each minion and returns with the dfined checks
- a python script that interacts with the salt peer interface to run the list_checks funciton in the nrpe module

The gen_icinga_conf.py script should be run on the icinga server, it publishes the command nrpe.list_checks (which the master must have configuration to allow it) and then spits out icinga configuration to stdout.


### The salt utility module for NRPE
> _modules/nrpe.py
    #!/usr/bin/env python
    '''
    Icinga NRPE utility functions for salt
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

    def _get_conf(fname='/etc/nagios/nrpe.cfg'):
        fp = salt.utils.fopen('/etc/nagios/nrpe.cfg', 'r')
        return fp.read()

    def run(name):
        '''
        Look in the conf for name and run the command configured. Return the retcode and output
        '''
        cmd = re.search(r'command\[%s]=(?P<command>.*)' %name, _get_conf())
        if cmd:
            command = cmd.groupdict()['command']
            return __salt__['cmd.run_all'](command)
        else:
            return 'Command with name "%s" not found' %name

    def list_checks():
        ret = {}
        cmd = re.findall(r'^command\[(?P<name>.*?)]=(?P<command>.*)', _get_conf(), re.MULTILINE)
        for found in cmd:
            ret[found[0]] = found[1]
        return ret


### The salt local client

> gen_icinga_conf.py
    #!/usr/bin/env python
    # Import the Salt client library
    import salt.client

    def generate_conf():
        ''' Find all dommands in nrpe.cfg and generate the correct icinga conf '''

        template = '''
    define service {
        use                    generic-service
        host_name              %(hostname)s
        active_checks_enabled  0
        service_description    %(servicename)s
        check_command          check_nrpe_1arg!check_ntp
    }'''
        caller = salt.client.Caller()
        ret = caller.function('publish.publish', '*', 'nrpe.list_checks')
        conf = []
        for minion in sorted(ret):
            if type(ret[minion]) != type({}): continue # nrpe undefined on minion
            for servicename, command in ret[minion].items():
                mconf = template % {'hostname':minion, 'servicename': servicename}
                conf.append(mconf)
        print '\n'.join(conf)

    generate_conf()

