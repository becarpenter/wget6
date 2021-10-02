# Notes on making *wget* implement RFC6874bis

## The Patch
These notes assume you have read [draft-carpenter-6man-rfc6874bis](https://datatracker.ietf.org/doc/draft-carpenter-6man-rfc6874bis/). It describes how the **wget** utility has been made to support zone identifiers (interface names) in IPv6 literal addresses.

Firstly, this has only been done for Linux. I don't know how to build *wget* for Windows, and since relevant parts of the socket API are different for Windows, there might be more code to change.

For Linux only one small change was needed. In the module *host.c*, there is a Boolean function *is_valid_ipv6_address()* which is passed two pointers into a string, i.e. the beginning and end of the literal IPv6 address in a URL such as *http://[2001:db8::abcd]:80/whatever*.

It returns true if the address is well-formatted and false otherwise.

My patch is very simple: if the literal address is valid *and* followed by a '%' sign, the function returns *true*. (Unpatched, it returns *false*.) There is no need to check the validity of the zone identifier, because that will be checked when the code calls *getaddrinfo()*. Note that in the POSIX API, *getaddrinfo()* converts the *%zoneid* (if present) into an interface index as required for further socket calls. This just works, no other code changes are needed, and an invalid or missing zone identifier causes a clean error message. FYA, the added code in its entirety is

~~~~
      else if (ch == '%')
        {break;}
~~~~

(but it took me some hours studying the code to realise that was all I needed).

One peculiarity of *wget* is the module *url.c*, which uses the functions provided by *host.c*, but handles percent-encoding of URLs in a particular way. When it start analyzing a URL, it runs a function *reencode_escapes()* that percent-encodes any suitable characters, including % signs that are *not* immediately followed by two hex digits. In our case, that percent-encodes things like *[fe80::abcd%eth0]* as *[fe80::abcd%25eth0]*. But later -- after completing analysis of the URL -- it removes all percent-encoding (as it should, for transmission on the wire). Since this happens before calling *getaddrinfo()*, everything works as we want it to. A side effect of this is that (with my patch), *wget* does exactly what was suggested in RFC6487, which everybody said was too complicated.

Bottom line: patched *wget* works for both *[fe80::abcd%eth0]* and *[fe80::abcd%25eth0]*

## Examples

Here are several examples of how it works. The link-local address used is that of my home gateway, which includes a web server for management actions.

First here is what happens if you do not include an interface name:

~~~~
brian@brian-HP-14 ~ $ wget http://[fe80::2e3a:fdff:fea4:dde7]
--2021-10-01 16:26:57--  http://[fe80::2e3a:fdff:fea4:dde7]/
Connecting to [fe80::2e3a:fdff:fea4:dde7]:80... failed: Invalid argument.
~~~~
With unpatched *wget*, the next three would give that error again:

~~~~
brian@brian-HP-14 ~ $ wget http://[fe80::2e3a:fdff:fea4:dde7%wlp2s0]
--2021-10-01 16:25:17--  http://[fe80::2e3a:fdff:fea4:dde7%25wlp2s0]/
Connecting to fe80::2e3a:fdff:fea4:dde7%wlp2s0|fe80::2e3a:fdff:fea4:dde7|:80... connected.
HTTP request sent, awaiting response... 400 Bad Request
2021-10-01 16:25:17 ERROR 400: Bad Request.
~~~~
That worked -- it performed an HTTP request/response, but my home gateway doesn't like *wget* (it wants a modern browser).

~~~~
brian@brian-HP-14 ~ $ wget http://[fe80::2e3a:fdff:fea4:dde7%25wlp2s0]
--2021-10-01 16:26:08--  http://[fe80::2e3a:fdff:fea4:dde7%25wlp2s0]/
Connecting to fe80::2e3a:fdff:fea4:dde7%wlp2s0|fe80::2e3a:fdff:fea4:dde7|:80... connected.
HTTP request sent, awaiting response... 400 Bad Request
2021-10-01 16:26:08 ERROR 400: Bad Request.
~~~~
Same again, but find the subtle difference.

~~~~
brian@brian-HP-14 ~ $ wget https://[fe80::2e3a:fdff:fea4:dde7%wlp2s0]
--2021-10-01 16:06:44--  https://[fe80::2e3a:fdff:fea4:dde7%25wlp2s0]/
SSL_INIT
Connecting to fe80::2e3a:fdff:fea4:dde7%wlp2s0|fe80::2e3a:fdff:fea4:dde7|:443... connected.
ERROR: The certificate of ‘fe80::2e3a:fdff:fea4:dde7%wlp2s0’ is not trusted.
ERROR: The certificate of ‘fe80::2e3a:fdff:fea4:dde7%wlp2s0’ doesn't have a known issuer.
The certificate's owner does not match hostname ‘fe80::2e3a:fdff:fea4:dde7%wlp2s0’
~~~~
Similar, but we triggered TLS and it (correctly) found a certificate problem.

Now a few correctly detected error cases for completeness:

~~~~
$ wget http://[fe80::2e3a:fdff:fea4:dde7%enp4s0]
--2021-10-02 15:28:54--  http://[fe80::2e3a:fdff:fea4:dde7%25enp4s0]/
Connecting to fe80::2e3a:fdff:fea4:dde7%enp4s0|fe80::2e3a:fdff:fea4:dde7|:80... failed: Network is unreachable.

$ wget http://[fe80::2e3a:fdff:fea4:dde7%invalid]
--2021-10-02 15:29:47--  http://[fe80::2e3a:fdff:fea4:dde7%25invalid]/
failed: Name or service not known.
wget: unable to resolve host address ‘fe80::2e3a:fdff:fea4:dde7%invalid’

$ wget http://[fe80::2e3a:fdff:fea4:dde7%]
--2021-10-02 15:30:27--  http://[fe80::2e3a:fdff:fea4:dde7%25]/
Connecting to fe80::2e3a:fdff:fea4:dde7%|fe80::2e3a:fdff:fea4:dde7|:80... failed: Invalid argument.
~~~~

## Implementation notes:

On Linux you need to download the latest *wget* tarball (*wget-latest.tar.gz*). Expand it and cd to the *wget-1.21.2* (or similar) directory. 

Overwrite */src/host.c* with my version. Then it should be as simple as:

~~~~
   ./configure
   make
   make install
~~~~

I read the manual, but it didn't tell me that I needed to reboot Linux for that "install" to take effect.

Note some idefs that must be set (they were set by default on my system):

~~~~
   ENABLE_IPV6
   AI_NUMERICHOST
   HAVE_SOCKADDR_IN6_SCOPE_ID
~~~~
The last one is critical, of course.
