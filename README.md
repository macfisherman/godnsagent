godnsagent
============
godnsagent is a simple DNS server which downloads zones over HTTP(S) in JSON format,
parses them, stores them in memory and serves to clients.

* godnsagent listens on both TCP and UDP port 53.
* It does not support daemonization.
* There is no config file.
* There is no reading from local zone files.
* Malformed zones or records are ignored (with a warning in log).
* Logs to stdout.
* Uses many threads to handle connections (by Go goroutines).
* Exits gracefully on SIGINT or SIGTERM.

How to build
============
```bash
go get github.com/miekg/dns
git clone https://github.com/DevelopersPL/godnsagent.git
cd godnsagent
go build -ldflags "-X main.buildtime '`date`' -X main.buildcommit '`git log --pretty=format:'%h' -n 1`'"
```

How to run
============
* Parameter ```-z``` defines address of DNS zones. Defaults to http://localhost/zones.json
* Parameter ```-l``` defines the local IP (interface) to listen on. Defaults to all.
* Parameter ```-r``` enable recursive querying of specified servers for answers godnsagent can't provide itself.
* Parameter ```-k``` sets the API key (passed as GET or form value "key" to http notification handlers)
* Parameter ```-lf``` sets the logger flags (integer): https://golang.org/pkg/log/#pkg-constants

```
./godnsagent -z https://example.org/path/to/zones.json -l 127.0.0.1 -r 8.8.8.8:53 -k secretkeyhere
```

How it works
============
* Once you start the program, it will try to download the zones JSON document.
* If the download fails, the program will fail (exit with error code).
* It binds to ports 53 on TCP and UDP and serves queries.
* The longest matching zone is chosen.
* Answers are marked as authoritative.
* All NS records on the zone are returned with an answer as "Authoritative" section.
* If possible, resolutions for NS records are added as "Extra" section.
* An HTTP GET request to :5380/notify invokes a reload of zones (the reload fails gracefully)
* HTTP requests require valid key passed as a GET parameter if such a key is defined
* An HTTP POST request to :5380/notify/zones processes JSON body of request as zones (zones are merged with cache but contents are replaced)
* If recursive querying is enabled, the question will be forwarded to the specified server

Schema of zones file
============
* Class field is optional, defaults to IN
* Fields are case-insensitive
* TTL is optional, defaults to 3600
* Data must hold all information specific to record type (see MX, SOA, SRV, etc.)
* The zone should have SOA record, although godnsagent will not complain
* The zone should have NS records, although godnsagent will not complain
* Zone name (key) should be FQDN or godnsagent will make it FQDN
* Use FQDN whenever possible

```json
{
  "spigu.com.": [
    {"name": "spigu.com.", "type": "A", "tTl": 500, "data": "123.123.123.123"},
    {"name": "b.spigu.com.", "type": "A", "Class": "CH", "Ttl": 300, "data": "123.123.123.124"},
    {"name": "spigu.com", "type": "MX", "Class": "IN", "Ttl": 305, "data": "5 email.spigu.net."},
    {"name": "spigu.com", "type": "NS", "data": "marley.spigu.com."},
    {"name": "spigu.com", "type": "NS", "Class": "IN", "Ttl": 300, "data": "abc.spigu.com."},
    {"name": "spigu.com", "type": "SOA", "TTL": 300, "data": "abc.spigu.com. hostmaster.spigu.com. 1399838297 21600 3600 1814400 300"}
  ],
  "spigu.net.": [
    {"name": "spigu.net.", "type": "A", "tTl": 500, "data": "123.123.123.123"},
    {"name": "b.spigu.net.", "type": "A", "Class": "CH", "Ttl": 300, "data": "123.123.123.125"},
    {"name": "spigu.net", "type": "MX", "Class": "IN", "Ttl": 305, "data": "5 email.spigu.net."},
    {"name": "spigu.net", "type": "NS", "data": "marley.spigu.net."},
    {"name": "spigu.net", "type": "NS", "Class": "IN", "Ttl": 300, "data": "abc.spigu.net."},
    {"name": "abc.spigu.net", "type": "A", "data": "123.123.123.100"},
    {"name": "marley.spigu.net", "type": "A", "data": "123.123.123.101"},
    {"name": "spigu.net", "type": "SOA", "TTL": 300, "data": "marley.spigu.net. hostmaster.spigu.net. 1399838297 21600 3600 1814400 300"}
  ]
}
```

Acknowledgments
============
This software was created thanks to two amazing projects:
  * https://github.com/miekg/dns: DNS library by miekg provided awesome foundations of DNS in Go.
  * https://github.com/kenshinx/godns: Kenshinx's goDNS provided a great example and reference point for using it.
