A modern IRC experience
=======================

IRC is still the king for group chat in online communities, but it has a few problems when comparing to "modern" internet services. Historically, the most common solution to being connected and not miss messages is to have a CLI client (ircII, BitchXI, irssi, WeeChat) running on a computer always in a terminal multiplexer (screen, tmux) and then connect to that using SSH.

Today there exists multiple different ways to get a more modern experience. While web frontends for IRC are not a new thing, historically they were not able to provide a good experience. But in my opinion, web clients are now able to compete with CLI apps in ways of bringing a powerful and usable chat experience. 


There are many products and services available bringing IRC experience to the web. I have put togheter a list of popular services (that I know of): <http://irccloud.com>, <http://hipchat.com>, <http://grove.io>, <https://campfirenow.com/>, <https://github.com/kandanapp/kandan>). While these services can be great for many users, the problems I have are that they are either non-free (as in freedom), too pricey or the wrong fit if you aren't an organization, and some provide IRC like functionality without being IRC, which means you are chatting in a silo, or have limited interopability / flexibility.  

The good things these services have going for them is things like ease of use, good cross platform support (Android, iOS, etc), good user interfaces, embedding content (images, videos), better visual experience, and more. So I have a quest to bring some of these features to IRC using free software.

Another common way of being connected to IRC all the time is to use a *bouncer* (e.g. <http://wiki.znc.in/ZNC>) which also solves some of the same problems, but as bouncers commonly work using the IRC protocol itself, they struggle to achieve the same capabilities as WeeChat can provide using its own relay protocol. You can think of WeeChat + relay plugin as a super advanced and capable *bouncer* in addition to being a full client.

The software
------------

> [WeeChat][1] is a fast, light and extensible CLI chat client. It runs on many platforms like Linux, Unix, BSD, GNU Hurd, Mac OS X and Windows. It can support multiple protocols, is modular, has support for plugin in many languages (C, Python, Perl, Ruby, Lua, Tcl and Scheme), free software (GPLv3), strong and active community.

The biggest reason why we use WeeChat as backend client for a HTML5 frontend is that its relay plugin allows us to directly connect from the browser using Websockets. This means that the client does not need a special "backend service", as all that is provided by the IRC client itself.
  
> [Glowing bear][2] is a HTML5 web frontend for WeeChat that strives to be a modern and slick interface on top of WeeChat. It relies on WeeChat to do all the heavy lifting (connection, servers, history, etc) and then provides a few features on top of that, like content embedding (images, video) and desktop notification. It has also rudimentary support for running as a Firefox app or Chrome app, which means that it is really cross platform. It should run just as great on Android, IOS as on a desktop computer.


Screenshot
----------
Running as Chrome application in a separate window on Windows:

![Glowing bear screenshot][3]

Running as Firefox application on Android:

![Glowing bear android screenshot][4]

How to get started with WeeChat and Glowing Bear
-----------------

This guide assumes you already have a server available where you can run WeeChat permanently.


**How to get WeeChat connected to a server and be and ready for relay connection:**

  1. Log in to your shell
  2. Make sure weechat is installed, and make sure the version is at least **0.4.2** which is the lowest version supported by the web client. Check with command `weechat --version ` 
  3. Start  WeeChat in a screen to leave i permanently running. ` screen -S weechat weechat `.  Older versions of WeeChat uses the executable `weechat-curses` instead of `weechat`.
  4. Add a server. I will use Freenode as an example: `/server add freenode chat.freenode.net/6667 -autoconnect `
  5. Connect to IRC-server ` /connect freenode `
  6. Join a channel ` /join #weechat `
  7. Set up autojoin of channel ` /set irc.server.freenode.autojoin #weechat `
  8. Add a WeeChat relay that the web client will use  ` /relay add weechat 40900 ` You can choose any port you want instead of 40900.
  9. Set a password for relay clients  ` /set relay.network.password YOURPASSWORD `
  10. Save the settings you just entered ` /save `

**Configure WeeChat to be better**
  
  11. Enable many colors to use for nick coloring ` /set weechat.color.chat_nick_colors "22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,81,82,83,84,85,86,87,92,93,94,95,96,97,98,99,100,101,102,103,104,105,106,107,108,109,110,111,112,113,114,115,116,117,118,119,120,121,122,123,126,127,128,129,130,131,132,133,134,135,136,137,138,139,140,141,142,143,144,145,146,147,148,149,150,151,152,153,154,155,156,157,158,159,160,161,162,163,164,165,166,167,168,169,170,171,172,173,174,175,176,177,178,179,180,181,182,183,184,185,186,187,188,189,190,191,192,193,194,195,196,197,198,199,200,201" `
  12. Use colors in nicklist: ` /set irc.look.color_nicks_in_nicklist on  `
  13. Install script to get nick colors in chat lines too ` /script install colorize_nicks.py `
  14. Save the settings you just entered ` /save `

**Connect the web client**
 
  1. Navigate to the official hosted URL <http://glowing-bear.github.io/glowing-bear/>. (You can also clone the GitHub repository <https://github.com/glowing-bear/glowing-bear/> and host it yourself, it is only static files, no server setup required.)
  2. Enter the hostname of your server running WeeChat
  3. Enter the port you selected in the relay setup.
  4. Enter the password you chose in the relay setup.
  5. Click the usage instructions to get some information about key bindings.
  5. Connect!
  6. If you can't connect, check with command `/relay ` inside WeeChat to see incoming connection, if incoming connection is missing, check if firewalls are preventing access.
  
**Further reading**

  - WeeChat Quick Start : <http://weechat.org/files/doc/devel/weechat_quickstart.en.html>
  - WeeChat FAQ : <http://weechat.org/files/doc/weechat_faq.en.html>
  - WeeChat Script list : <http://weechat.org/scripts/>
  - WeeChat features with list of other front ends: <http://weechat.org/about/features/>

Please join `#glowing-bear` on Freenode if you want to chat with the developers of glowing bear.

Please join `#weechat` on Freenode if you want to chat with the developers of WeeChat.


  
  [1]: http://weechat.org
  [2]: http://cormier.github.io/glowing-bear/
  [3]: http://hveem.no/ss/weechat-web-client720.png
  [4]: http://hveem.no/ss/gb-mobile-new.png
