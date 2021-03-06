![exitmap logo](https://nullhypothesis.github.com/exitmap_logo.png)
===================================================================

[![Build Status](https://travis-ci.org/NullHypothesis/exitmap.svg?branch=master)](https://travis-ci.org/NullHypothesis/exitmap)

Overview
--------

Exitmap is a fast and modular Python-based scanner for
[Tor](https://www.torproject.org) exit relays.  Exitmap modules implement tasks
that are run over (a subset of) all exit relays.  If you have a background in
functional programming, think of exitmap as a `map()` interface for Tor exit
relays: Modules can perform any TCP-based networking task like fetching a web
page, uploading a file, connecting to an SSH server, or joining an IRC channel.

In practice, exitmap is useful to monitor the reliability and trustworthiness of
exit relays.  The Tor Project uses exitmap to check for false negatives on the
Tor Project's [check](https://check.torproject.org) service and to find
[malicious exit relays](http://www.cs.kau.se/philwint/spoiled_onions).  It is
easy to develop new modules for exitmap; just have a look at the file HACKING in
the doc/ directory or check out one of the existing modules.

Exitmap uses [Stem](https://stem.torproject.org) to create circuits to all given
exit relays.  Each time tor notifies exitmap of an established circuit, a module
is invoked for the newly established circuit.  Modules can be pure Python
scripts or executables.  For executables,
[torsocks](https://github.com/dgoulet/torsocks/) is necessary.

Finally, note that exitmap is a network measurement tool and of little use to
ordinary Tor users.  The Tor Project is already running the tool regularly.
More exitmap scans just cause unnecessary network load.  The only reason exitmap
is publicly available is because its source code and design might be of interest
to some.

Installation
------------

Exitmap uses the library Stem to communicate with Tor.  There are
[plenty of ways](https://stem.torproject.org/download.html) to install Stem.
The easiest might be to use pip in combination with the provided
requirements.txt file:

    $ pip install -r requirements.txt

Running exitmap
---------------

The only argument exitmap requires is the name of a module.  For example, you
can run exitmap with the checktest module by running:

    $ ./bin/exitmap checktest

The command line output will then show you how Tor bootstraps, the output of the
checktest module, and a scan summary.  If you don't need three hops and prefer
to use two hops with a static first hop, run:

    $ ./bin/exitmap --first-hop 1234567890ABCDEF1234567890ABCDEF12345678 checktest

To run the same test over German exit relays only, execute:

    $ ./bin/exitmap --country DE --first-hop 1234567890ABCDEF1234567890ABCDEF12345678 checktest

If you want to pause for five seconds in between circuit creations to reduce the
load on the Tor network and the scanning destination, run:

    $ ./bin/exitmap --build-delay 5 checktest

Note that `1234567890ABCDEF1234567890ABCDEF12345678` is a pseudo fingerprint
that you should replace with an exit relay that you control.

To learn more about all of exitmap's options, run:

    $ ./bin/exitmap --help

Exitmap comes with batteries included, providing the following modules:

* testfds: Tests if an exit relay is able to fetch the content of a simple
  web page.  If an exit relay is unable to do that, it might not have enough
  file descriptors available.
* checktest: Attempts to find false negatives in the Tor Project's
  [check](https://check.torproject.org) service.
* dnspoison: Attempts to resolve several domains and compares the received DNS A
  records to the expected records.
* dnssec: Detects exit relays whose resolver does not validate DNSSEC.
* patchingCheck: Checks for file tampering.
* cloudflared: Checks if a web site returns a CloudFlare CAPTCHA.
* rtt: Measure round-trip times through an exit to various destinations.

Configuration
-------------

By default, exitmap tries to read the file .exitmaprc in your home directory.
The file accepts all command line options, but you have to replace minuses with
underscores.  Here is an example:

    [Defaults]
    first_hop = 1234567890ABCDEF1234567890ABCDEF12345678
    verbosity = debug
    build_delay = 1
    analysis_dir = /path/to/exitmap_scans

Advanced Binary Integrity Test Using the Tor Browser
----------------------------------------------------

To run the advancedBinaryIntegrity module, which runs tests over the Tor
Browser, you need the following software:

* tor
* python2.7
* python-stem
* python-beautifulsoup
* python-mechanize
* python-requests
* python-dnspython
* python-psutil

And the following additional python modules which provide the mozilla interface
for marionette:

* marionette_driver
* mozdevice
* mozfile
* mozinfo
* mozlog
* mozprocess
* mozprofile
* mozrunner
* mozversion

Unfortunately, some dependency packages require python 2.7 and are
incompatible with python 3.

Under Debian GNU/Linux, you can install the required software with:

    $ sudo apt install tor python2.7 python-stem python-beautifulsoup python-mechanize python-requests python-dnspython python-psutil

And the python modules with:

    $ pip install -r requirements-abi.txt

Next you need to download and unpack the Tor Browser software. The Tor Browser
controlling module was only tested with `tor-browser_en-US` version.
Change the location of the `tb_dir` parameter in the module source accordlingly.

Several modifications of the Tor Browser are necessary.

Start the Tor Browser and change the following things:

* open firefox settings, and set Downloads to always download to the default location.
* download some executable (eg. random installer from `filehippo.com`) and check the checkbox and click yes when asked for downloading an external file
* install the following addons:
  - https://addons.mozilla.org/en-US/firefox/addon/http-request-logger
  - https://addons.mozilla.org/en-US/firefox/addon/enable-rightclick-and-copy/
  - https://addons.mozilla.org/en-US/firefox/addon/ublock-origin/
  http request logger is needed to find new POST requests for mail injection
  ublock is required because the download buttons on the websites can't be clicked for some reason if ads are displayed
  enable rightclick is needed to disallow websites from preventing right-clicks on the website
* after installation of the ublock addon, go to download.cnet.com and click the button to permanently disable strict filtering for the site by ublock.
* try to download an installer on cnet.com and do the same "permanent disable" for their CDN (dw.cbsi.com)
* copy the mimeTypes.rdf file over the existing one in the folder
  `tor-browser_en-US/Browser/TorBrowser/Data/Browser/profile.default/`
  to make executables be auto-downloaded as well
* create an empty file
  `tor-browser_en-US/Browser/Desktop/http-request-log.txt`
  eg. to do that under GNU/Linux, run in the shell:

    touch <your torbrowser path>/tor-browser_en-US/Browser/Desktop/http-request-log.txt

Then run the advancedBinaryIntegrity module like a normal module. It
auto-starts the Tor Browser, runs tests and closes the Browser again.
Scanning results are written to several files (one fingerprint per line):

* successfully checked (unmodified binary): `success.txt`
* were malicious (modified binary): `malicious.txt`
* failed: `failed-fps.txt`
* ran into timeout somewhere `timeout-fps.txt`

It is recommended to split the list of Tor exits to scan into multiple
sub-lists, especially if you want to scan all exits. Sometimes you have to
re-try some exit nodes later or several times, because of timeouts and
irregular throughput rates.

You could for example take the `success.txt` file to generate a new list of exit nodes by running

    grep -Fvxf success.txt Exitlist > Exitlist_new


Alternatives
------------

Don't like exitmap?  Then have a look at
[tortunnel](http://www.thoughtcrime.org/software/tortunnel/),
[SoaT](https://gitweb.torproject.org/torflow.git/tree/NetworkScanners/ExitAuthority/README.ExitScanning),
[torscanner](https://code.google.com/p/torscanner/),
[DetecTor](http://detector.io/DetecTor.html), or
[SelekTOR](http://www.dazzleships.net/selektor-for-linux/).

Tests
-----

Before submitting pull requests, please make sure that all unit tests pass by
running:

    $ pip install -r requirements-dev.txt
    $ py.test --cov-report term-missing --cov-config .coveragerc --cov=src test

Feedback
--------

Contact: Philipp Winter <phw@nymity.ch>  
OpenPGP fingerprint: `B369 E7A2 18FE CEAD EB96  8C73 CF70 89E3 D7FD C0D0`
