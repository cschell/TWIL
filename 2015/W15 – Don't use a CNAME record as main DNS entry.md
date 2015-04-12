# Don't use a CNAME record as main DNS entry

Last week I changed the main record of my domain from A-Records with fix IPs to CNAME-Records pointing to another domain. So instead of having something like

    $ dig chrisschell.de
    chrisschell.de.		1800	IN	A	176.9.162.44

[dig](https://en.wikipedia.org/wiki/Dig_(command)) returned
    
    $ dig chrisschell.de
    chrisschell.de.		1800	IN	CNAME	mywebspace.example.com
    mywebspace.example.com. 3600	IN	A	176.9.162.44

I had successfully verified this change in advance on a test domain and have therefor been pretty confident. A request for `chrisschell.de` returned a CNAME to `mywebspace.example.com` which returned the correct ip – everything's fine!

But little did I know \*diabolic laughter\*.

After getting unusually few mails it dawned me that there might be something off. And indeed, requesting my domain's MX record resulted in an empty answer:

    $ dig mx chrisschell.de
    chrisschell.de.		1800	IN	CNAME	mywebspace.example.com
    mywebspace.example.com.		IN	MX

If someone tried to send me an email, his server would try to forward the mail to the mailserver listed for `mywebspace.example.com`, not `chrisschell.de`. Since the destination domain has no MX-record set, mailservers got an empty answer and I didn't get any mails.

After switching back to status quo I did some digging and found the RFC for ["Common DNS Operational and Configuration Errors"](https://tools.ietf.org/html/rfc1912#page-6), which had the following to say to me:

> A CNAME record is not allowed to coexist with any other data.

and

> This is often attempted by inexperienced administrators as an obvious
   way to allow your domain name to also be a host.

Damn.
