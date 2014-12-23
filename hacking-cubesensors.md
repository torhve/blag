Intercepting Cubesensors sensor data
====================================

So I purchased these [CubeSensors](https://cubesensors.com/) a while ago to monitor my home environment. As many other geeks I like playing around with numbers and using tools to visualize them in the manner I please. When the package from them (finally) arrived I was a bit let down to discover they didn't have a good API to get the raw data from them. They did however provide access to a beta API which I requested access to.

![What a cube looks like](http://blog.cubesensors.com/wp-content/uploads/2014/11/mycubes-home.jpeg)

Preferably, in my opinion, the base station should come with a web server with a documented API that you can extract numbers from directly on the LAN. This means you could use the devices when you aren't even connected to the internet and allows for greater flexibility in general.

After poking around a bit on my network I discovered from the MAC-address of the base station's network card that the base station is a [Raspberry PI](http://www.raspberrypi.org/help/what-is-a-raspberry-pi/) with a custom card attached that handles the [ZigBee protocol](http://en.wikipedia.org/wiki/ZigBee) the base station uses to connect to the cubes. At this point I was tempted to unscrew the screws and detach the storage card from the Raspberry PI and try to get access to the box and inject my own software into the data stream so I could extract the raw numbers and put them into my own database. Instead I started a network sniffer to see what the base station is up to.

I discovered a few things:

 - It runs [Debian](https://www.debian.org/)!
 - It uses ssh to bitbucket.org to update itself (I think).
 - It posts JSON data to data.cubesensors.com once per minute.

The data looks like this:

    POST /v1/devices/base-station-id-redacted HTTP/1.1..Host: data.cubesensors.com..Accept-Encoding: identity..Content-Length: 464..Content-T
    ype: application/json....{"firmware": 59, "data": [{"cube": "cubeid-redacted", "temp": 2199, "voc": 3098, "battery": 2376, "ligh
    t": 3364, "firmware": 59, "humidity": 3339, "pressure": 1006, "voc_resistance": 75082, "noisedba": 47, "rssi": -63, "charging": 3
    }, {"cube": "cube-id-redacted", "temp": 1600, "voc": 450, "battery": 2400, "light": 4094, "firmware": 59, "humidity": 4470, "pres
    sure": 1006, "voc_resistance": 7904, "noisedba": 43, "rssi": -80, "charging": 3}], "time": 1419099276}

Then I had the idea for another solution to get what I wanted, to set up a web proxy that read that JSON data and manipulate it as I best see fit before it travels over the series of tubes to the cube cloud.

For this purpose I will use a special bundle of the popular web server Nginx called [OpenResty](http://openresty.org/) which is my weapon of choice for these type of tasks.

Here's the snippet of configuration that I needed to get a nginx to think it's data.cubesensors.com

> `nginx.conf`

    # use google resolver to resolve the real cubesensors host using DNS.
    resolver 8.8.8.8;
    server {
        ...
        # instruct nginx who it is.
        server_name  data.cubesensors.com;
        ...
        # all the API requests starts with v1, this might have to be updated later if they change their API URLs
        location /v1/ {
            # For safety we only allow requests from the real device
            allow base.station.ip.network/24;
            # Or localhost
            allow 127.0.0.1;
            deny all;
            # This is the fun part, we hook the Nginx access handler to intercept the request before it travels over the wire.
            access_by_lua_file '/home/cubesensor/intercept.lua';
            # After Lua had its turn, lets just proxy the original request, so we can also send the data to cubesensors to use their app.
            proxy_pass http://data.cubesensors.com/v1/;
        }
    }

To trick the base station to send data to my web proxy I activated this iptables firewall rule:
 
    iptables -t nat -I PREROUTING -d data.cubesensors.com -p tcp --dport 80 -i eth1 -j DNAT --to-destination my.web.server:80

Then I had to write the `intercept.lua` to parse the JSON data and save it to my PostgreSQL database.

> Here's the critical  `intercept.lua` parts

    -- Read the full body from the request
    ngx.req.read_body()
    -- get the JSON
    local body = ngx.req.get_body_data()
    local success, jdata = pcall(function()
      return cjson.decode(body)
    end)
    if success then
    local success, err = pcall(function()
      local data = jdata.data
      if data then
        local time = jdata.time
        for _, cube in pairs(data) do
          local id = cube.cube
          -- Delete a few keys we don't want to store in the SQL
          cube.cube = nil
          cube.firmware = nil
          cube.charging = nil

        local keys = {}
        local values = {}

        for key, val in pairs(cube) do
            -- Algos from http://www.visionect.com/blog/raspberry-pi-e-paper/
            table.insert(keys, key)
            if key == 'voc' then
                val = math.max(val - 900, 0)*0.4 + math.min(val, 900)
            elseif key == 'light' then
                val = 10/6.0*(1+(val/1024.0)*4.787*math.exp(-(math.pow((val-2048)/400.0+1, 2)/50.0))) * (102400.0/math.max(15, val) - 25)
            elseif key == 'humidity' then
                val = val/100
            end
            table.insert(values, val)
        end
        keys[#keys + 1] = 'time'
        keys = table.concat(keys, ',')
        values[#values + 1] = "date_trunc('minute', to_timestamp(" .. tostring(time) .. "))"
        values = table.concat(values, ',')
        local sql = [[
            INSERT INTO
            data_]] .. id .. [[
            (]] .. keys .. [[)
            VALUES (]] .. values .. [[)
        ]]
        dbreq(sql)
      end
    end
    end)

The raw numbers were a bit strange, so I did some googling and found [this blog post](http://www.visionect.com/blog/raspberry-pi-e-paper/) that apparently got some math that cubesensors need to transform them into human readable numbers. I feel like CubeSensors should provide this information somewhere, or maybe they do, but I could not find anything anywhere. The transformation is done in the interceptor before saving them to SQL so there's less work for the API later.

Now that I have access to the raw numbers I can easily build my own frontends and easily access them however I please. One of the ways I created to display the data is a HTML5 app that looks like this:

![Only showing parts of the graphs](http://hveem.no/ss/cubeapp.png)

What I need now is something to automatically open  windows to let in fresh air when it gets too bad 😜

You can find complete sources if you want to have a go yourself at [http://github.com/torhve/cubesensor](http://github.com/torhve/cubesensor)

Enjoy!

