Getting started with Lua web development using docker as your Lua web application server
========================================================================================


In this blog post I will guide you through a path to getting you started with Lua web development by using the container technology software [Docker][1].

Why [Docker][1] ?
-----------------

From their web page:
> Docker is an open-source project to easily create lightweight, portable, self-sufficient containers from any application. The same container that a developer builds and tests on a laptop can run at scale, in production, on VMs, bare metal, OpenStack clusters, public clouds and more. 

This means that we have an excellent way to reproduce and contain a development setup for the Lua project we will be creating that will not mess up your system the way most application package managers like npm, luarocks, virtualenv + easy_install/pip likes to do. Another major drawback with most application package managers are that they are not good at creating isolated enviroments, so you will have version conflicts or breakage if application1 wants a different version of a library than application2. Or like invirtualenv's case it will still be tied to the system in some ways.

The goal of this guide is to end up in this type of setup:

    Internets => nginx:80
                  |--> docker (app1):8080 
                  |       |--> nginx+lua: 8080
                  |--> docker (app2):8090
                  |       |--> nginx+lua: 8090
                  ...

The point is that we use [Docker][1] the way we normally would use an app server like uwsgi, gunicorn, nodejs, php5-fpm or similar.

Why [Openresty][2] ?
--------------------

A downside with running Lua(jit) inside your Nginx is that this is not enabled by default in most distributions of Nginx, and since Nginx does not (yet) support dynamic loading of modules which means a recompile of Nginx is needed to get support for running Lua. And while Nginx recompilation is really fast, and it while it supports hot code reload, it is still painful because of security and maintainability considerations.

To get Lua support in Nginx we will be using the Nginx-bundle called [Openresty][2]. Openresty comes bundled with many Nginx and Lua modules useful for web development.

### Building our Dockerfile


A Dockerfile is:
> Docker can act as a builder and read instructions from a text Dockerfile to automate the steps you would otherwise take manually to create an image. Executing docker build will run your steps and commit them along the way, giving you a final image

We will create a Dockerfile which sets up a Ubuntu with [Openresty][2].

Installation help for Ubuntu will be provided, if you use a different Linux distribution please find instructions here: <http://www.docker.io/gettingstarted/#h_installation>

Installation of [Docker][1] on Ubuntu with kernel 3.8 or newer:

    sudo curl https://get.docker.io/gpg | sudo apt-key add -

    sudo sh -c "echo deb http://get.docker.io/ubuntu docker main\
    echo deb "echo deb http://get.docker.io/ubuntu docker main" | sudo tee /etc/apt/sources.d/docker.list
    sudo apt-get update
    sudo apt-get install lxc-docker
    
### Create a project directory, with logdir, nginx.conf and Dockerfile

    mkdir helloproj
    mkdir helloproj/logs
    cd helloproj
    
    cat <<EOF > Dockerfile
    # Dockerfile for openresty
    # VERSION   0.0.1

    FROM ubuntu:12.04
    MAINTAINER Tor Hveem <tor@hveem.no>
    ENV REFRESHED_AT 2013-12-12

    RUN    echo "deb-src http://archive.ubuntu.com/ubuntu precise main" >> /etc/apt/sources.list
    RUN    sed -i /etc/apt/sources.list 's/main$/main universe/'
    RUN    apt-get update
    RUN    apt-get -y upgrade
    RUN    apt-get -y install wget vim git libpq-dev

    # Openresty (Nginx)
    RUN    apt-get -y build-dep nginx
    RUN    wget http://openresty.org/download/ngx_openresty-1.4.3.9.tar.gz
    RUN    tar xvfz ngx_openresty-1.4.3.9.tar.gz
    RUN    cd ngx_openresty-1.4.3.9 ; ./configure --with-luajit  --with-http_addition_module --with-http_dav_module --with-http_geoip_module --with-http_gzip_static_module --with-http_image_filter_module --with-http_realip_module --with-http_stub_status_module --with-http_ssl_module --with-http_sub_module --with-http_xslt_module --with-ipv6 --with-http_postgres_module --with-pcre-jit;  make ; make install

    EXPOSE 8080
    CMD /usr/local/openresty/nginx/sbin/nginx -p `pwd` -c nginx.conf

    EOF
    
### Build the image using our Dockerfile
    
    sudo docker build -t="torhve/openresty" .
This will create the contained environment which we then can re-use and launch for many projects later. The `-t` flag is the name for the image, you can choose your own name here if you want, to help you refer to the image by name later. The name mostly matters if you want to share/submit your docker image to repositories, see: <http://docs.docker.io/en/latest/use/workingwithrepository/> for more information about that.


### Create a simple nginx.conf for the project

    cat <<EOF > nginx.conf
    worker_processes 1;
    error_log stderr notice;
    daemon off;
    events {
        worker_connections 1024;
    }

    http {
        variables_hash_max_size 1024;
        access_log off;
        include /usr/local/openresty/nginx/conf/mime.types;
        set_real_ip_from 127.0.0.1/8;
        real_ip_header X-Real-IP;
        charset utf-8;

        server {
            listen 8080;
            lua_code_cache off;

            location / {
                default_type text/html;
                content_by_lua_file "app.lua";
            }

            location /static/ {
                alias static/;
            }
        }
    }

    EOF
    
Now we can start the docker image that we built. We will map the directory from the host to the container so you can continue to use your favorite editor and development environment from the host. Note that the nginx.conf and app lives outside the container, so you can re-use this container image for all of your lua projects.

### Run our newly created Docker image

    sudo docker run -t -i -p 8080:8080 -v=`pwd`:/helloproj -w=/helloproj torhve/openresty 
`-p` expose the container port 8080 to the host port 8080.
`-v` is the shared directory.
`-w` is the working directoriy inside the container.
`-t` and `-i` for interactive tty.
`torhve/openresty` is the name of the image. List images with `sudo docker images`

### Create the Hello World app.lua

    cat <<EOF > app.lua
    ngx.say('Hello World!')
    ngx.exit(200)
    EOF

The nginx web server should now be serving content, lets confirm with `curl`:

    curl http://127.0.0.1:8080/
Which should print:
    
    Hello World!

*Awesome!*

### Deploy


To serve content to the internets let me show you to configure Nginx running on the host to get content from our helloproj Docker image. You could easily have two (or more!) running Docker process on different ports, one with `lua_code_cache on` on one port for production, and one without it for development. Here is a simplistic nginx conf that will proxy the connections to the Docker image:

    cat <<EOF > /etc/nginx/sites-enabled/helloproj.conf
    server {
        listen 80;
       
        location / {
            proxy_pass http://127.0.0.1:8080/;
            proxy_set_header  X-Real-IP  $remote_addr;
        }
    }
    EOF
        
Basically we just proxy every request to the container using the built in proxy feature of Nginx. We also use set the real IP header to the clients IP address so the contained Nginx process can see the client's IP address. 

To extend/improve this setup you could set the root directory to be the same directory as the helloproj and then serve static content using the host nginx, and let the contained nginx only worry about dynamic content. It could also be extended to cache requests using one of the many Nginx caching techniques.

To start and keep your containers running you can use many different techniques, some viable solutions are systemd, supervisord, upstart, or just a good old initscript.

### Potential pitfalls

Running two Nginxes will have an impact on performance compared to running a single app on a single front facing Nginx, because every request will be proxied. This is similar to what many app servers will also have to do in some ways, but usually one of the advantages of Lua (and php!) is that you can run your application directly inside the webserver without any proxying. So for instance if you have an application that handles large uploads you need to think about how you want to handle that. 


The source for this guide is available in a [GitHub repository][3]. Please fork and send changes if you have suggestions or improvements.


  [1]: http://docker.io/ "Docker"
  [2]: http://openresty.org/
  [3]: https://github.com/torhve/openresty-docker
