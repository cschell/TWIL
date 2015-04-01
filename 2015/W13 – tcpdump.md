# Rediscovering `tcpdump`

This week I rediscovered `tcpdump`. Like Wireshark, this command displays your network traffic. Unlike Wireshark, it’s a shell command and is therefor a more convenient choice for server admins who rely on ssh.

The easiest way to use tcpdump is to just run `sudo tcpdump` from your terminal.  Depending on your current traffic it will flood your screen with every request going out or coming in.

To see what’s in those packages you can use `-A`:

    sudo tcpdump -A port 25 or port 587
    
This will show you what’s going on when you send an e-mail from your local machine.

Another interesting usecase is analyzing HTML logins:

1. run `sudo tcpdump -A dst host runnable.com`
2. visit a [dummy login](http://runnable.com/VRxDxTW9TKMwmEcG/output/) I’ve made on [runnable.com](runnable.com)
3. submit random data
4. watch your tcpdump output
5. see, [why it’s a good idea to use SSL](http://get.chrisschell.de/10N2D)

Of course you can do much more with this tool and luckily there is a lot of documentation out there. Some interesting links:

- http://www.tecmint.com/12-tcpdump-commands-a-network-sniffer-tool/
- https://danielmiessler.com/study/tcpdump/#common
