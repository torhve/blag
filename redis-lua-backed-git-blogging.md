# Simple blog platform backed by lua, redis, nginx and git

Previously I wrote about [my simple blog software that used git as database](/simple-blogging-with-lua-and-git). But after a while I found myself wanting not to ruin LuaJIT's fantastic speed by spawning git commands for every visitor. And since I was using [Redis](http://redis.io/) for a silly visitor counter already it seemed like a waste not to leverage it in some way. [@Graaten](http://twitter.com/graaten) had this brilliant idea to have a git hook that generates the data needed by the blog upon commit. 

So I set out to write a simple bash [Git](http://git-scm.com/) post hook that turns the list of [Markdown](http://en.wikipedia.org/wiki/Markdown)-files into a JSON list of commit logs and then feed that into Redis with the redis-cli command.

## The Git Hook

The shell script is fairly well commented so it should be self explanatory :)

> md/.git/hooks/post-commit

    #!/bin/bash
    # Author: Tor Hveem

    # Loop through each markdown file
    for FILE in $(ls *md) ; do
        # strip file extension
        TITLE="${FILE%%.*}"
        REDIS_KEY="post:$TITLE:log" 
        # get json log output with unix timestamp date
        OUTPUT=$(exec git log --pretty=format:"'%h': {'commit': '%H','author': '%an <%ae>',  'timestamp': %ct,  'message': '%s'}" -- $FILE)
        # Switch linesbreaks to , with goal to eventually make a list
        OUTPUT=$(echo $OUTPUT | tr "\n" ",")
        # Strip last ,
        OUTPUT="${OUTPUT%%,}"
        # Json list
        OUTPUT="[$OUTPUT]"
        # Feed the output into redis
        echo "SET $REDIS_KEY \"$OUTPUT\"" | redis-cli > /dev/null
        # Find first timestamp of post
        TIMESTAMP=$(exec git log --pretty=format:"%ct" --reverse -- $FILE | head -1)
        # Add the post to the sorted set of posts
        echo "ZADD posts $TIMESTAMP $TITLE" | redis-cli > /dev/null
    done

## The Lua blog code

I'm using the Redis datatype of Sorted Set for the post list. With the title as the member variable and the date as the score it's very easy to poke the Redis db for a list of posts sorted by date.

The function to get list of blog posts with date is as simple as this:
> index.lua

    --[[ 
    Return a table with post date as key and title as val
    ]]
    local function posts_with_dates(limit)
        local posts, err = red:zrevrange('posts', 0, limit, 'withscores')
        if err then return {} end
        posts = red:array_to_hash(posts)
        return swap(posts)
    end

The single post view is also much more simple and faster now. I run the redis zscore function to get the date of the title sent in using query path. If the redis command fails I simply return 404.
Or if it gets a date back, it opens the file and compiles the markdown.

> index.lua

    local page = match[1] 
    local date, err = red:zscore('posts', page)
    -- No match, return 404
    if err or date == ngx.null then
        return ngx.HTTP_NOT_FOUND
    end
    local mdfile =  BLAGDIR .. page .. '.md'
    local mdfilefp = assert(io.open(mdfile, 'r'))
    local mdcontent = mdfilefp:read('*a')
    local mdhtml = markdown(mdcontent) 

The next steps for increasing performance now will be to either have markdown generated on git commit too, or have it generated on first visit and then saved to redis for all the next visit. That method would need check in the commit log json data for any updates and thus needing regeneration.

## Source
You can find the source at [my github repository for this project](https://github.com/torhve/LuaWeb).











