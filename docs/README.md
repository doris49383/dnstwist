![dnstwist](/docs/dnstwist.png)
===============================

See what sort of trouble users can get in trying to type your domain name.
Find lookalike domains that adversaries can use to attack you. Can detect
typosquatters, phishing attacks, fraud, and brand impersonation. Useful as an
additional source of targeted threat intelligence.

![Demo](/docs/demo.gif)

DNS fuzzing is an automated workflow for discovering potentially malicious
domains targeting your organisation. This tool works by generating a large list
of permutations based on a domain name you provide and then checking if any of
those permutations are in use.
Additionally, it can generate fuzzy hashes of the web pages to see if they are
part of an ongoing phishing attack or brand impersonation, and much more!

In a hurry? Try it in your web browser: [dnstwist.it](https://dnstwist.it)


Key features
------------

- Variety of highly effective domain fuzzing algorithms
- Unicode domain names (IDN)
- Additional domain permutations from dictionary files
- Efficient multithreaded task distribution
- Live phishing webpage detection:
  - HTML similarity with fuzzy hashes (ssdeep)
  - Screenshot visual similarity with perceptual hashes (pHash)
- Rogue MX host detection (intercepting misdirected e-mails)
- GeoIP location
- Export to CSV and JSON


Installation
------------

**Python PIP**

```
$ pip install dnstwist[full]
```

**Git**

If you want to run the latest version of the code, you can install it from Git:

```
$ git clone https://github.com/elceef/dnstwist.git
$ cd dnstwist
$ pip install .
```

**Debian/Ubuntu/Kali Linux**

Invoke the following command to install the tool with all extra packages:

```
$ sudo apt install dnstwist
```

**Fedora Linux**

```
$ sudo dnf install dnstwist
```

**macOS**

This will install `dnstwist` along with all dependencies, and the binary will
be added to `$PATH`.

```
$ brew install dnstwist
```

**Docker**

Pull and run official image from the Docker Hub:

```
$ docker run -it elceef/dnstwist
```


Quick start guide
-----------------

The tool will run the provided domain name through its fuzzing algorithms and
generate a list of potential phishing domains along with DNS records.

Usually thousands of domain permutations are generated - especially for longer
input domains. In such cases, it may be practical to display only the ones that
are registered:

```
$ dnstwist --registered domain.name
```

Ensure your DNS server can handle thousands of requests within a short period
of time. Otherwise, you can specify an external DNS or DNS-over-HTTPS server
with `--nameservers` argument.

Manually checking each domain name in terms of serving a phishing site might be
time-consuming. To address this, `dnstwist` makes use of so-called fuzzy hashes
(context triggered piecewise hashes, often called ssdeep) and perceptual hashes
(pHash). Fuzzy hashing is a concept that involves the ability to compare two
inputs (HTML code) and determine a fundamental level of similarity, while
perceptual hash is a fingerprint dervied from visual features of an image
(web browser screenshot).

The unique feature of detecting similar HTML source code can be enabled with
`--ssdeep` argument. For each generated domain, `dnstwist` will fetch content
from responding HTTP server (following possible redirects), normalize HTML code
and compare its fuzzy hash with the one for the original (initial) domain. The
level of similarity will be expressed as a percentage.

Important: In cases when the effective URL is the same as for the original
domain, the fuzzy hash is not calculated at all in order to reject false
positive indications.

Note: Keep in mind it's rather unlikely to get 100% match, even for MITM attack
frameworks, and that a phishing site can have completely different HTML
source code. However, each notification is a strong indicator and should be
inspected carefully regardless of the score.

```
$ dnstwist --ssdeep domain.name
```

In some cases, phishing sites are served from a specific URL. If you provide a
full or partial URL address as an argument, `dnstwist` will parse it and apply
for each generated domain name variant. Use `--ssdeep-url` to override URL to
fetch the original web page from.

```
$ dnstwist --ssdeep https://domain.name/owa/
$ dnstwist --ssdeep --ssdeep-url https://different.domain/owa/ domain.name
```

Additionally, if Chromium browser is installed, `dnstwist` can run it in
headless mode to render web pages, take screenshots and calculate pHash to
evaluate visual similarity expressed as percentage.

```
$ dnstwist --phash domain.name
```

Web page screenshots in PNG format can be saved to specified location:

```
$ dnstwist --phash --screenshots /tmp/domain domain.name
```

Sometimes attackers set up e-mail honey pots on phishing domains and wait for
mistyped e-mails to arrive. In this scenario, attackers would configure their
server to vacuum up all e-mail addressed to that domain, regardless of the user
it was sent towards. Another `dnstwist` feature allows performing a simple test
on each mail server (advertised through DNS MX record) to check which one can
be used for such hostile intent. Suspicious servers will be flagged with
`SPYING-MX` string.

Note: Be aware of possible false positives. Some mail servers only pretend to
accept incorrectly addressed e-mails but then discard those messages.

```
$ dnstwist --mxcheck domain.name
```

If domain permutations generated by the fuzzing algorithms are insufficient,
please supply `dnstwist` with a dictionary file. Some dictionary samples with
a list of the most common words used in phishing campaigns are included.

```
$ dnstwist --dictionary dictionaries/english.dict domain.name
```

If you need to check whether domains with different TLD exist, just supply
a dictionary file with the list of TLD.

```
$ dnstwist --tld dictionaries/common_tlds.dict example.com
```

Apart from the colorful terminal output, the tool allows exporting results to
CSV and JSON. In case you need just the permutations without making any DNS
lookups, use `--format list` argument:

```
$ dnstwist --format csv domain.name | column -t -s,
$ dnstwist --format json domain.name | jq
$ dnstwist --format list domain.name
```

The tool can perform real-time lookups to return geographical location
(approximated to the country) of IPv4 addresses.

```
$ dnstwist --geoip domain.name
```

The GeoIP2 library is used by default. Country database location has to be
specified with `$GEOLITE2_MMDB` environment variable. If the library or the
database are not present, the tool will fall-back to the older GeoIP Legacy.

To display all available options with brief descriptions simply execute the
tool without any arguments.

Happy hunting!


API
---

In case you need to consume the data produced by the tool within your code,
probably the most convenient and fast way is to pass the input as follows.

```
>>> import dnstwist
>>> data = dnstwist.run(domain='domain.name', registered=True, format='null')
```

The arguments for `dnstwist.run()` are translated internally, so the usage is
very similar to the command line. Keep in mind that `dnstwist.run()` spawns
a number of daemon threads.


Notes on coverage
-----------------

Along with the length of the domain, the number of variants generated by the
algorithms increases considerably, and therefore the time and resources needed
to verify them. It's mathematically impossible to check all domain
permutations - especially for longer input domains which would require millions
of DNS lookups.
For this reason, this tool generates and checks domains very close to the
original one. Theoretically, these are the most attractive domains from the
attacker's point of view. However, be aware that the imagination of the
aggressors is unlimited.

Unicode tables consist of thousands of characters with many of them visually
similar to each other. However, despite the fact certain characters are
encodable using punycode, most TLD authorities will reject them during domain
registration process. In general, TLD authorities disallow mixing of characters
coming from different Unicode scripts or maintain their own sets of acceptable
characters. With that being said, the homoglyph fuzzer was build on top of
carefully researched range of Unicode characters (homoglyphs) to ensure that
generated domains can be registered in practice.


It really works
---------------

The scanner is utilized by tens of SOC and incident response teams around the
globe, as well as independent information security analysts and researchers.
On top of this, it's integrated into products and services of many security
providers, in particular but not only:

Splunk ESCU, RecordedFuture, SpiderFoot, DigitalShadows, SecurityRisk,
SmartFense, ThreatPipes, PaloAlto Cortex XSOAR, Rapid7 InsightConnect SOAR,
Mimecast, Watcher, Intel Owl, PatrOwl, VDA Labs, Appsecco.


Contact
-------

To send questions, thoughts or a bar of chocolate, just drop an e-mail at
[marcin@ulikowski.pl](mailto:marcin@ulikowski.pl).
Any feedback is appreciated. If you have found some confirmed phishing domains
or just like this tool, please don't hesitate and send a message. Thank you.
