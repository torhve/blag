How I created a minimalistic blog platform with lua
======

My motivation was just having a simple no frills blog for publishing some of my latest writings. I have also worked a bit with Wordpress lately, and I wanted to play with a comepletely different software stack. This project is not very serious and is meant for personal use.
The most unusual part about this project is it's usage of nginx as the "app server". This is possible using lua, since there is a nginx lua module that enables you to call lua from the nginx conf. Have a look at [openresty's site](http://openresty.org) for more about what it enables you to do.

Let me begin by listing the components being used:

##### Components

-  [Lua(jit)](http://luajit.org/luajit.html) Superfast and lean scripting language.
-  [Nginx](http://nginx.org/) Superfast and lean web server.
-  [Openresty](http://openresty.org/) A bundle of plugins for nginx letting me run lua code directly in Nginx. 
-  [Redis](http://redis.io/) In-memory database with disk persist. 
-  [Git](http://git-scm.com/) Disitributed version control system.
-  [Markdown](http://en.wikipedia.org/wiki/Markdown) Lightweight markup language.


Now we know the components in use, so now I can explain how they work togheter. Lua runs inside nginx and is being used as backend as it is being called in the web developer world. There is little to no javascript running in this setup. The backend loads a few templates (header, footer, etc) using Zed Shaw's tiny templating engine from another micro framework in Lua, check it out here: [Tir Microframework](http://sheddingbikes.com/posts/1289384533.html)

The index template looks in the predefined git repository for a list for markdown files, these will be the blog posts, then it runs git log to figure out the date the blog post was created and displays a list.

The blog post template extracts a filename from the URL and loads the corresponding Markdown file. The markdown gets compiled with Niklas Frykholm's Markdown parser written in lua.

Each view has a simple counter in redis, with the primary motivation of showcasing the speed of these pages. The Redis database is not really needed for any functionality.

Publishing a new article is then just the small matter of writing a Markdown file and pushing it to the blog repository and then it will display in the list and have its own permanent URL.

### Show me some code!

> index.lua (with some parts edited out for clarity)
    -- load our template engine
    local tirtemplate = require('tirtemplate')
    -- Load redis
    local redis = require "resty.redis"
    local markdown = require "markdown"

    -- Set the content type
    ngx.header.content_type = 'text/html';

    -- use nginx $root variable for template dir, needs trailing slash
    TEMPLATEDIR = ngx.var.root .. 'luanew/';
    -- The git repository storing the markdown files. Needs trailing slash
    BLAGDIR = TEMPLATEDIR .. 'md/'
    BLAGTITLE = 'hveem.no'

    -- the db global
    red = nil

    -- Get all the files in a dir
    local function get_posts()
        local directory = shell_escape(BLAGDIR)
        local i, t, popen = 0, {}, io.popen
        local handle = popen('ls "'..directory..'"')
        if not handle then return {} end
        for filename in handle:lines() do
            i = i + 1
            t[i] = filename
        end
        handle:close()
        return t
    end

    -- Find the commit dates for  a file in a git dir was
    local function file2gitci(dir, filename)
        local i, t, popen = 0, {}, io.popen
        local dir, filename = shell_escape(dir), shell_escape(filename)
        local cmd = 'git --git-dir "'..dir..'.git" log --pretty=format:"%ct" --date=local --reverse -- "'..filename..'"'
        local handle = popen(cmd)
        for gitdate in handle:lines() do
            i = i + 1
            t[i] = gitdate
        end
        handle:close()
        return t
    end

    local function filename2title(filename)
        title = filename:gsub('.md$', ''):gsub('-', ' ')
        return title
    end

    function slugify(title)
        slug = title:gsub(' ', '-')
        return slug
    end

    --- Better safe than sorry
    function shell_escape(s)
        return (tostring(s) or ''):gsub('"', '\\"')
    end

    -- 
    -- Index view
    --
    local function index()
        
        -- increment index counter
        local counter, err = red:incr("index_visist_counter")

        local postlist = get_posts()
        local posts = {}
        for i, post in pairs(postlist) do
            local gitdate = file2gitci(BLAGDIR, post)
            -- Skip unversioned files
            if #gitdate > 0 then 
                -- Use first date
                posts[gitdate[1]] = filename2title(post)
            end
        end
        -- Sort on timestamp key
        table.sort(posts, function(a,b) return tonumber(a)>tonumber(b) end)

        -- load template
        local page = tirtemplate.tload('index.html')
        local context = {
            title = BLAGTITLE, 
            counter = tostring(counter),
            posts = posts,
        }
        -- render template with counter as context
        -- and return it to nginx
        ngx.print( page(context) )
    end

    --
    -- blog view for a single post
    --
    local function blog(match)
        local page = match[1] 
        local mdfiles = get_posts()
        local mdcurrent = nil
        for i, mdfile in pairs(mdfiles) do
            if page..'.md' == mdfile then
                mdcurrent = mdfile
                break
            end
        end
        -- No match, return 404
        if not mdcurrent then
            return ngx.HTTP_NOT_FOUND
        end
        
        local mdfile =  BLAGDIR .. mdcurrent
        local mdfilefp = assert(io.open(mdfile, 'r'))
        local mdcontent = mdfilefp:read('*a')
        mdfilefp:close()
        local mdhtml = markdown(mdcontent) 
        local gitdate = file2gitci(BLAGDIR, mdcurrent)
        -- increment visist counter
        local counter, err = red:incr(page..":visit")

        local ctx = {
            created = ngx.http_time(gitdate[1]),
            content = mdhtml,
            counter = counter,
        } 
        local template = tirtemplate.tload('blog.html')
        ngx.print( template(ctx) )

    end

    -- 
    -- Initialise db
    --
    local function init_db()
        -- Start redis connection
        red = redis:new()
        local ok, err = red:connect("unix:/var/run/redis/redis.sock")
        if not ok then
            ngx.say("failed to connect: ", err)
            return
        end
    end

    --
    -- End db, we could close here, but park it in the pool instead
    --
    local function end_db()
        -- put it into the connection pool of size 100,
        -- with 0 idle timeout
        local ok, err = red:set_keepalive(0, 100)
        if not ok then
            ngx.say("failed to set keepalive: ", err)
            return
        end
    end

    -- mapping patterns to views
    local routes = {
        ['$']         = index,
        ['(.*)$']     = blog,
    }

    local BASE = '/'
    -- iterate route patterns and find view
    for pattern, view in pairs(routes) do
        local uri = '^' .. BASE .. pattern
        local match = ngx.re.match(ngx.var.uri, uri, "") -- regex mather in compile mode
        if match then
            init_db()
            exit = view(match) or ngx.HTTP_OK
            end_db()
            ngx.exit( exit )
        end
    end
    -- no match, return 404
    ngx.exit( ngx.HTTP_NOT_FOUND )

> nginx.conf, the Nginx web server configuration

    lua_package_path '/home/www/lua/?.lua;;';
    server {
      listen 80;
      server_name example.no www.example.no;
      set $root /home/www/;
      root $root;

      # Serve static if file exist, or send to lua
      location / { try_files $uri @lua; }
      # Lua app
      location @lua {
          content_by_lua_file $root/lua/index.lua;
      }
    }              

Read the [followup](/redis-lua-backed-git-blogging), where I change the backend a bit to use redis.

## Source
You can find the source at [my github repository for this project](https://github.com/torhve/LuaWeb).


