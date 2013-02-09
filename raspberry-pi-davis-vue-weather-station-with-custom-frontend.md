# My weather station project

This page is the documentation for how I set up my weather station with my home made front end for modern HTML5 visualizations.
The reason for building my own frontend is simply that I find all the free software for weather displays very lacking in presentation and features. I built my own frontend with goals of being responsive, so both small (mobile) and huge screens would get pretty graphs. All the layout is thus responsive design, with SVG vectorized graphics for the visualizations. There's also some JavaScript to detect screen size and make some adjumestments to the page to get it all to work smoothly on all screen sizes. The frontend is still very much a work in progress, but supports the most critical features of displaying live current weather and different graphs of historic data, with records.

*Note:* This document is a work in progress and will have a few weak parts until I have worked out a few kinks, etc.

## Prequisites

*    **Raspbery PI** for fetching data from weather station, grabbing pictures from web camera and pushing to the web.
*    **A weaher station** (This guide is using Davis Vantage Vue)
*    **A web and database server host** (With a public reacable address, running Ubuntu/Nginx/OpenResty/Postgresql)
*    **A web camera**, for still images and timelapse (I bought a Microsoft LifeCam Studio HD)

## Components

*   Raspbian
*   Openvpn for secure transfer to web/database server
*   Weewx
*   some python glue
*   my weather frontend in lua+postgresl [AmatYr](http://github.com/torhve/amatyr) running on a web server

## Installation

The guide is a bit terse and written in a style that expects familiarity with linux, and will only list the specifics.

#### Log in to raspberry PI
Install all the dependencies. Some of these are only extra deps required of weewx if you intend to use its own weather page generation. I believe you can skip cheetah, imaging and pyephem.

    sudo apt-get install python-configobj 
    sudo apt-get install python-serial
    sudo apt-get install python-cheetah 
    sudo apt-get install python-imaging 
    sudo apt-get install python-psycopg2

    sudo apt-get install python-dev
    sudo apt-get install python-pip
    sudo pip install pyephem

#### Install weewx

Find the latest wersion from weewx download page.

    wget -O weewx.tar.gz http://sourceforge.net/projects/weewx/files/weewx-2.1.1.tar.gz/download
    tar xvfz weewx.tar.gz
    cd weewx*
    sudo ./setup.py install

Then download my postgresql patch and apply it.

    wget http://hveem.no/weewx-2.1.1.postgresql.patch ; patch -p1 < weewx-2.1.1.postgresql.patch

#### Configure weewx

    cd /home/weewx
    sudo chown -R pi /home/weewx 

Configure weewx to match your needs and wants. I use metric system, and disable every weewx's own generation tools.

    vim weewx.conf

We also need our specific database setup for amatyr frontend. Note that my patch for weewx for postgresql is at this point incomplate and only supports the archive database. I am in the process of figuring out the statsdb and getting it to work with postgresql.

    [StdArchive]
    archive_database = archive_psql
    stats_database = stats_psql

    [[archive_psql]]
        host = 10.9.36.1
        user = wwex
        password = wwexwwex
        database = weewx
        driver = weedb.postgresql

    [[stats_psql]]
        host = 10.9.36.1
        user = weewx
        password = wwexwwex
        database = stats
        driver = weedb.postgresql

You can do additonal davis specific settings with the next command, like setting altimeter, correct date/time, etc.
The important part for me is to set the archive interval to 60 seconds to log to database every 60 seconds.

    ./bin/config_vp.py weewx.conf --help
    ./bin/config_vp.py weewx.conf --set-interval 60

Configure weewx to start on boot

    sudo cp /home/weewx/start_scripts/Debian/weewx /etc/init.d 
    sudo chmod +x /etc/init.d/weewx 
    sudo update-rc.d weewx defaults 98 
    sudo /etc/init.d/weewx start

#### Install and configure openvpn server on the web server

    apt-get install openvpn
    mkdir /etc/openvpn/keys
    openvpn --genkey --secret /etc/openvpn/keys/raspberry.key

    cat <<EOF >> /etc/openvpn/raspberry.conf
    dev tun
    ifconfig 10.9.36.1 10.9.36.2
    secret keys/raspberry.key
    keepalive 1 60
    ping-timer-rem
    persist-tun
    persist-key
    comp-lzo
    fast-io
    EOF

#### Install and configure openvpn client on the raspberry

    sudo apt-get install openvpn
    sudo mkdir /etc/openvpn/keys

Transfer the static key from the web server

    sudo scp -p web-server-ip:/etc/openvpn/keys/raspberry.key /etc/openvpn/keys/raspberry.key

Set client conf

    cat <<EOF >> /etc/openvpn/raspberry.conf
    remote web-server-ip
    dev tun
    ifconfig 10.9.36.2 10.9.36.1
    secret keys/raspberry.key
    keepalive 1 60
    ping-timer-rem
    persist-tun
    persist-key
    comp-lzo
    nobind
    float
    fast-io
    EOF

#### Install postgresql database on the webserver

    sudo apt-get install postgresql-server
    sudo -u postgresql psql postgres
    create user wwex with password 'wwexwwex';
    alter role wwex createdb;

Alter posgresql.conf to listen to IP
    
    listen_addresses = '17.0.0.1,10.9.36.1'        # what IP address(es) to listen on;

Alter pga_hba.conf to allow connection for IP

    host all   all 10.9.36.2 md5

#### Start weewx and check that it's logging OK

You should see successful pushes to database in the logfile after you run these commands:

    service weewx start
    tail -f /var/log/messages

#### Install [AmatYr](http://github.com/torhve/amatyr) on the web server

Read its own [Readme](https://github.com/torhve/Amatyr/blob/master/README.md) over at github. Fair warning, project is still in a bit of flux so expect some errors in the doc.

## A few pictures of the setup

###### My dad installing the weather sensore suite
![The weather station sensor](http://hveem.no/davis.jpg)
###### The wireless antenna for sending the signal to the internets
![The wireless antenna to send the signal back home](http://hveem.no/antenne.jpg)
###### The Raspberry PI connected to a wireless repeater, a web camera and the weather station console
![The weather station console and raspberry pi](http://hveem.no/weatherconsole.jpg)

You can visit my own weather site at [yr.hveem.no](http://yr.hveem.no/)


