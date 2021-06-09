
[![Build Status](https://travis-ci.org/proxytunnel/proxytunnel.svg?branch=master)](https://travis-ci.org/proxytunnel/proxytunnel)

# Proxytunnel

Author:		Jos Visser <josv@osp.nl>, Mark Janssen <maniac@maniac.nl>

ProxyTunnel Introduction

ProxyTunnel is a program that connects stdin and stdout to a server somewhere on the network, through a standard HTTPS proxy. We mostly use it to tunnel SSH sessions through HTTP(S) proxies, allowing us to do many things that wouldn't be possible without ProxyTunnel.

Proxytunnel can currently do the following:

    Create tunnels using HTTP and HTTPS proxies (That understand the HTTP CONNECT command).
    Work as a back-end driver for an OpenSSH client, and create SSH connections through HTTP(S) proxies.
    Work as a stand-alone application, listening on a port for connections, and then tunneling these connections to a specified destination. 

If you want to make effective use of ProxyTunnel, the proxy server you are going to be tunneling through must adhere to some requirements.

    Must support HTTP CONNECT command
    Must allow you to connect to destination machine and host, with or without HTTP proxy authentication 

Most proxies however only allow connections to a number of pre-defined ports. These ports usually include 80(http) and 443(https). Some other proxies also allow traffic on other ports or ranges. Try to find out what ports your proxy allows you to connect through. Your best guess is to test for 80 and 443, and then check for some other common ports like 8000, 8080, 8081, 8082 and the nntp ports 119(nntp) and 563(snntp).

If you have figured out what ports your proxy allows you to connect through, the fun can start. If it allows you to connect to a port you want access too, like the pop3 or imap ports you are in luck, since you can now set up a direct tunnel to these servers and read mail for example. Usually however you will be stuck with access to only port 80 and 443 (like we were, when we wrote ProxyTunnel). To be able to get access to more then just these ports you need access to a server on the internet where you are able to log-in via SSH on one of these ports. In our case we set up the SSH daemons on our home and office systems to listen to port 443(https), since these weren't used (and 80 was), and the port was allowed by the firewall/proxy.

After having setup a SSH daemon on an accessable port, we configured our local SSH clients to use ProxyTunnel as a back-end to make the connection. Doing this involves creating a ~/.ssh/config file, specifying a host-alias there, and telling SSH to use a proxy-command, using the ProxyCommand statement and our ProxyTunnel tool to do it.

We now have access to SSH on our 'unrestricted' system on the internet. As you may know, SSH allows you to do port-forwarding and other nice tricks. Using this knowledge it is possible to forward and port anywhere. I myself usually setup some port-forwardings for my mail (2 imap tunnels) and usenet. But i'm sure you can think up of many things you'd like to connect to. Now you can.


# Build docs is [here](INSTALL.md) 

# Usage (man page is [here](docs/proxytunnel.1.adoc)):

Proxytunnel is very easy to use, when running proxytunnel with the help
option it specifies it's command-line options.

```
$ ./proxytunnel --help
proxytunnel 1.9.9 Copyright 2001-2018 Proxytunnel Project
Usage: proxytunnel [OPTIONS]...
Build generic tunnels through HTTPS proxies using HTTP authentication

Standard options:
 -i, --inetd                Run from inetd (default: off)
 -a, --standalone=INT       Run as standalone daemon on specified port
 -p, --proxy=STRING         Local proxy host:port combination
 -r, --remproxy=STRING      Remote proxy host:port combination (using 2 proxies)
 -d, --dest=STRING          Destination host:port combination
 -e, --encrypt              SSL encrypt data between local proxy and destination
 -E, --encrypt-proxy        SSL encrypt data between client and local proxy
 -X, --encrypt-remproxy     SSL encrypt data between local and remote proxy
 -W, --wa-bug-29744         workaround ASF Bugzilla 29744, if SSL is active stop
                            using it after CONNECT (might not work on all setups;
                            see /usr/share/doc/proxytunnel/README.Debian.gz)
 -B, --buggy-encrypt-proxy  Equivalent to -E -W, provided for backwards
                            compatibility
 -L                         (legacy) enforce TLSv1 connection
 -T, --no-ssl3              Do not connect using SSLv3

Additional options for specific features:
 -z, --no-check-certificate Don't verify server SSL certificate
 -C, --cacert=STRING        Path to trusted CA certificate or directory
 -F, --passfile=STRING      File with credentials for proxy authentication
 -P, --proxyauth=STRING     Proxy auth credentials user:pass combination
 -R, --remproxyauth=STRING  Remote proxy auth credentials user:pass combination
 -N, --ntlm                 Use NTLM based authentication
 -t, --domain=STRING        NTLM domain (default: autodetect)
 -H, --header=STRING        Add additional HTTP headers to send to proxy
 -o STRING                  send custom Host Header
 -x, --proctitle=STRING     Use a different process title

Miscellaneous options:
 -v, --verbose              Turn on verbosity
 -q, --quiet                Suppress messages
 -h, --help                 Print help and exit
 -V, --version              Print version and exit
```

To use this program with OpenSSH to connect to a host somewhere, create
a $HOME/.ssh/config file with the following content:

```
Host foobar
	ProtocolKeepAlives 30
	ProxyCommand /path/to/proxytunnel -p proxy:8080 -P username
-d mybox.athome.nl:443
```

With:

```
- foobar		The symbolic name of the host you want to connect to
- proxy			The host name of the proxy you want to connect through
- 8080			The port number where the proxy software listens to
- username		Your proxy userid (password will be prompted)
- mybox.athome.nl	The hostname of the box you want to connect to (ultimately)
- 443			The port number of the SSH daemon on mybox.athome.nl
```

If your proxy doesn't require the username and password for using it,
you can skip these options. If you don't provide the password on the
command-line (which is recommended) you will be prompted for it by
proxytunnel. If you are on a trusted system you can also put the
password in an environment variable, and tell proxytunnel where to
find it with '-S'.

If you want to run proxytunnel from inetd add the '--inetd' option.

Most HTTPS proxies do not allow access to ports other than 443 (HTTPS)
and 563 (SNEWS), so some hacking is necessary to start the SSH daemon on
the required port. (On the server side add an extra Port statement in
the sshd_config file, or use a redirect rule in your firewall.)

When your proxy uses NTLM authentication (like Microsoft IIS proxy)
you need to specify -N to enable NTLM, and then specify your username
and password (and optionally domain, if autodetection fails).
The NT domain can be specified on the commandline if the
auto-detection doesn't work for you (which is usually doesn't)

If you want to have the first proxy connect to another http proxy (like
one you can control, specify -r proxy2:port. The first proxy will then
connect to this remote proxy, which will be asked to connect to the 
requested destination. Note that authentication doesn't (yet) work on
this remote proxy. For more information regarding this feature, check
out http://dag.wieers.com/howto/ssh-http-tunneling/

If your proxy is more advanced, and does protocol inspection it will
detect that your connection is not a real HTTPS/SSL connection. You
can enable SSL encryption (using -e), which will work around this
problem, however, you need to setup stunnel4 on the other side, or
connect to a process that understands SSL itself.

When all this is in place, execute an "ssh foobar" and you're in business!

# Environment Variables

Proxytunnel can make use of the following environment variables:

```
PROXYUSER		Username for the proxy-authentication
PROXYPASS		Password for the proxy-authentication
REMPROXYUSER		Username for remote proxy-authentication
REMPROXYPASS		Password for remote proxy-authentication
HTTP_PROXY		Primary proxy host and port information
			Format: HTTP_PROXY=http://<host>:<port>/
```

# Authentication File

Proxytunnel can read authentication data from a file (-F/--passfile)

The format for this file is:
```
<field> = <value>
<field> = <value>
etc
```

One entry per line, 1 space before and after the equal sign.

The accepted fields are:
 * proxy_user
 * proxy_passwd
 * remproxy_user
 * remproxy_passwd

Share and Enjoy!

Jos Visser <josv@osp.nl>
Mark Janssen <maniac@maniac.nl>
