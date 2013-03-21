# Salt Stack XMPP Gateway 

Salt is very *open ended*. That is one attribute I love in the software I am using. One dictionary defines open ended like this:

1. Not restrained by definite limits, restrictions, or structure.
2. Allowing for or adaptable to change.
3. Inconclusive or indefinite
4. Allowing for a spontaneous, unstructured response: an open-ended question.

Except for that third point, that is pretty much the Salt Stack definition right there. Sorry for sounding like a salesman, but after using under 2 hours to build a simple XMPP gateway to control possibly thousands of minions, allow me to feel a little giddy.

Let me demonstrate, here we go!


## Prosody, a lightweight XMPP-server in Lua

I have used Prosody in a couple of projects earlier, it's very light weight and easy to work with, it will serve perfectly for this proof of concept.

Install prosody
     apt-get install prosody
Configure prosody
> /etc/prosody/prosody.cfg.lua                        

         admins = { 'admin@salt.demo.no' }
        "watchregistrations"; -- Alert admins of registrations
    allow_registration = true;
    VirtualHost "salt.demo.no"

Restart prosody
    service prosody restart

This demo uses two XMPP-users, one admin-account that will control the other account, used by the gateway.

Add admin account, set a password
    prosodyctl adduser admin@salt.demo.no
Add salt account, set a passord
    prosodyctl adduser salt@salt.demo.no

At this point you can log in with your XMPP client to admin@salt.demo.no to see if XMPP server is working.

## Salt API

Salt API is a REST API for Salt, allowing you to interface with Salt over HTTP. For this demo I could have used python local-client directly. But using Salt API allows for separation of XMPP and salt, say if you want to build agents for thousands for minions, or just have nice separation.

Configure salt api (check Salt API docs for details)
> /etc/salt/master

    rest_cherrypy:
      port: 8000
      ssl_crt: /etc/pki/tls/certs/localhost.crt
      ssl_key: /etc/pki/tls/certs/localhost.key
      debug: True 

    external_auth:
       pam:
         xmppuser:
           - test.*
           - status.*

Start salt-api
    salt-api

We allow the XMPP-gateway user to use two salt modules: test and status (e.g. test.version, status.uptime)

Add the user xmppuser:
    adduser xmppuser

Install XMPP library for python
    apt-get install python-xmpp

## The Salt XMPP gateway script

*Note*: This is just a demo, and not meant for production usage. 
The gateway script fires up a master bot and 1 minion bot for every minion. It will start off by trying to register a new account in-band for every minion and then send a message to the specified admin reporting in that it's ready for duty.
The gateway has one command *minions* to list all the minions. All other text is treated as a function, e.g. *test.version*
If you talk to a minion, it will run the command for that minion, if you talk to the master, it will run command on all minions.

Screenshots below!

### Installation

First adapt my configuration into your needs and save it to the file config.yaml:

> config.yaml
    saltapiurl: 'http://localhost:8000/'  
    saltuser: 'xmppuser'                  
    saltpass: '73Hengebruer'         
    xmppadminuser: 'admin@salt.demo.no' 
    stripdomain: '.demo.no'             
    username: 'salt@salt.demo.no'       
    password: '66Kjensleuttrykket'      


Then I wrote a simple Salt REST API python client. It's inspired by the pepper utility that will eventually do that, but it's not finished yet.

> saltrest.py

    import urllib
    import urllib2
    from cookielib import CookieJar
    import json

    HEADERS = {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
    }
    cj = CookieJar()
    class MyHTTPRedirectHandler(urllib2.HTTPRedirectHandler):
        def http_error_302(self, req, fp, code, msg, headers):
            # patch headers to include salt token
            token = headers['X-Auth-Token']
            HEADERS['X-Auth-Token'] = token
            return urllib2.HTTPRedirectHandler.http_error_302(self, req, fp, code, msg, headers)

        http_error_301 = http_error_303 = http_error_307 = http_error_302
    cookieprocessor = urllib2.HTTPCookieProcessor(cj)

    opener = urllib2.build_opener(MyHTTPRedirectHandler, cookieprocessor)
    urllib2.install_opener(opener)

    class SaltREST(object):

        def __init__(self, config):
            self.config = config
            self.login()

        def login(self):
            lowstate_login =[{
                'eauth': 'pam',
                'username': self.config['saltuser'],
                'password': self.config['saltpass'],
            }]
            postdata = json.dumps(lowstate_login).encode()

            req = urllib2.Request(self.config['saltapiurl']+'login', postdata, HEADERS)
            f = urllib2.urlopen(req)
            return f.read()

        def get_minions(self):

            lowstate = [{
                'client': 'local',
                'tgt': '*',
                'fun': 'test.version',
            }]

            postdata = json.dumps(lowstate).encode()
            req = urllib2.Request(self.config['saltapiurl']+'', postdata, HEADERS)
            f = urllib2.urlopen(req)
            ret = json.loads(f.read())
            # Format minions as a list with minion FQDNs
            ret = [x.replace(self.config['stripdomain'], '') for x in ret['return'][0].keys()]
            return ret

        def call(self, lowstate):
            postdata = json.dumps(lowstate).encode()
            req = urllib2.Request(self.config['saltapiurl'], postdata, HEADERS)
            f = urllib2.urlopen(req)
            ret = json.loads(f.read())
            return ret

This is the botherder script:

> salt-xmpp.py

    import sys
    import os
    import xmpp
    from threading import Thread, Event
    import yaml

# Our Salt REST API
    import saltrest

# flag to tell all threads to stop
    _stop = Event()


    def single_node_xmpp_outputter(ret):
        ret = ret['return'][0]
        fret = ''
        for host, val in ret.items():
            fret += '%s\n' %val
        return fret

    def xmpp_outputter(ret):
        ret = ret['return'][0]
        fret = ''
        for host, val in ret.items():
            fret += '%s: %s\n' %(host.replace(CONFIG['stripdomain'], ''), val)
        return fret


    def masterMessageCB(conn, mess):
        text=mess.getBody()
        user=mess.getFrom()
        jid = xmpp.protocol.JID(user).getStripped()
        print 'Got command:', text
        if jid == CONFIG['xmppadminuser']:
            if text == 'minions':
                # make a nice list
                conn.send(xmpp.Message(mess.getFrom(), ', '.join(MINIONS)))
            else:
                lowstate = [{
                    'client': 'local',
                    'tgt': '*',
                    'fun': text,
                }]
                ret = xmpp_outputter(salt.call(lowstate))
                conn.send(xmpp.Message(mess.getFrom(), ret) )


    def make_msg_handler(tgt):
        def minionCB(dispatcher, mess):
            print '[%s] %s' % (dispatcher._owner.Resource, mess)
            text=mess.getBody()
            user=mess.getFrom()
            jid = xmpp.protocol.JID(user).getStripped()
            if jid == CONFIG['xmppadminuser']:
                lowstate = [{
                    'client': 'local',
                    'tgt': tgt + CONFIG['stripdomain'],
                    'fun': text,
                }]
                ret = single_node_xmpp_outputter(salt.call(lowstate))
                dispatcher.send(xmpp.Message(mess.getFrom(), ret) )
        return minionCB


    def startminion(username, password):
        jid=xmpp.protocol.JID(username)
        cli=xmpp.Client(jid.getDomain(), debug=False)
        cli.connect()


        should_register = True
        if should_register:
            # getRegInfo has a bug that puts the username as a direct child of the
            # IQ, instead of inside the query element.  The below will work, but
            # won't return an error when the user is known, however the register
            # call will return the error.
            xmpp.features.getRegInfo(cli,
                                     jid.getDomain(),
                                     #{'username':jid.getNode()},
                                     sync=True)

            if xmpp.features.register(cli,
                                      jid.getDomain(),
                                      {'username':jid.getNode(),
                                       'password':password}):
                sys.stderr.write("Successfully register: %s!\n" %jid.getNode())
            else:
                sys.stderr.write("Error while registering: %s\n" %jid.getNode())

        authres=cli.auth(jid.getNode(),password)
        if not authres:
            print "Unable to authorize %s - check login/password." %jid.getNode()
            return None
            #sys.exit(1)
        if authres<>'sasl':
            print "Warning: unable to perform SASL auth.  Old authentication method used!"
        cli.RegisterHandler('message', make_msg_handler(jid.getNode()))
        cli.sendInitPresence()
        cli.send(xmpp.protocol.Message(CONFIG['xmppadminuser'],'Hello, Salt minion %s reporting for duty.' %jid.getNode()))

        return cli
            
    def startmaster(username, password):

        jid=xmpp.protocol.JID(username)
        cli=xmpp.Client(jid.getDomain(), debug=False)
        cli.connect()

        authres=cli.auth(jid.getNode(),password)
        if not authres:
            print "Unable to authorize - check login/password."
            sys.exit(1)
        if authres<>'sasl':
            print "Warning: unable to perform SASL auth.  Old authentication method used!"
        cli.RegisterHandler('message', masterMessageCB)
        cli.sendInitPresence()
        cli.send(xmpp.protocol.Message(CONFIG['xmppadminuser'],'Salt gateway ready for action.'))
        return cli
        
    def process_until_disconnect(bot):
        ret = -1
        while ret != 0 and not _stop.is_set():
            ret = bot.Process(1)


    root = os.path.dirname(os.path.abspath(__file__))
    CONFIG = yaml.safe_load(file(root+'/config.yaml').read())
    salt = saltrest.SaltREST(CONFIG)
# Get minions so we can create bots, uses test.ping to get minion list
    MINIONS = salt.get_minions()
    username = CONFIG['username']
    password = CONFIG['password']
    _stop.clear()

# Start master
    masterbot = startmaster(username, password)
    try:
        Thread(target=process_until_disconnect, args=(masterbot,)).start()
        for minion in MINIONS:
            minionbot = startminion(minion+'@salt.idrift.no', 'sharedbotpwfordemo')
            if minionbot:
                Thread(target=process_until_disconnect, args=(minionbot,)).start()
        # Block main thread waiting for KeyboardInterrupt
        #while True:
        #    pass
    except KeyboardInterrupt:
        _stop.set()
        print "Bye!"



     
##### Jitsi client showing the chat session with the Salt XMPP gateway
![Master chat](http://hveem.no/ss/salt-xmpp.png)
##### Jitsi client showing the chat session with a salt minion
![Minion chat](http://hveem.no/ss/salt-xmpp-minion.png)

Source: <https://github.com/torhve/salt-xmpp>
