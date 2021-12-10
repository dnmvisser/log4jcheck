# Northwave Log4j CVE-2021-44228 checker

Friday 10 December 2021 a new Proof-of-Concept [1] addressing a Remote code Execution (RCE) vulnerability in the Java library 'log4j' [2] was published. This vulnerability has not been disclosed to the developers of the software upfront. The vulnerability is being tracked as CVE-2021-44228 [3]. More information on the vulnerability can be found in the Northwave Threat response [4]

Northwave created a testing script that checks for vulnerable systems using injection of the payload in the User-Agent header and as a part of a HTTP GET request. Vulnerable systems are detected by listening for incoming DNS requests that contain a UUID specically created for the target. By listening for incoming DNS instead of deploying (for example) an LDAP server, we increase the likelyhood that vulnerable systems can be detected that have outbound traffic filtering in place. In practice, outbound DNS is often allowed. 

## DISCLAIMER
Note that the script only performs 2 specific checks:User Agent and HTTP GET request. This will cause false negatives in cases where other headers, specific input fields, etc. need to be targeted to trigger the vulnerability. Feel free to add extra checks to the script.

## Setting up a DNS server

First, we need a subdomain that we can use to receive incoming DNS requests. In this case we use the zone `log4jdnsreq.northwave.nl` and we deploy our script on `log4jchecker.northwave.nl`. Configure a DNS entry as follows:

```
log4jdnsreq 3600 IN  NS log4jchecker.northwave.nl.
```

We now set up a BIND DNS server on a Debian system using `apt install bind9` and add the following to the `/etc/bind/named.conf.options` file:

```
	recursion no;
    allow-transfer { none; };
```

This disables recusing as we do not want to run an open DNS server. Configure logging in `/etc/bind/named.conf.local` by adding the following configuration:

```
logging {
	channel querylog {
		file "log4jdnsreq";
		severity debug 3;
		print-time yes;
	};
	category queries { querylog;};
};
```
Don't forget to restart BIND using `systemctl restart bind9`. Check if the logging works by performing a DNS query for `xyz.log4jdnsreq.northwave.nl`. One or more queries should show up in 

## Running the script

Install any Python dependencies using `pip install -r requirements.txt`. Edit the script to change the following line to the DNS zone you configured:

```
HOSTNAME = "log4jdnsreq.northwave.nl"
```

You can now run the script using the following syntax:

```
python3 nw_log4jcheck.py https://www.northwave.nl
```

The last line of the output shows if the system was found to be vulnerable:

```
INFO:root:NOT VULNERABLE! No incoming DNS request to 3414db71-309a-4288-83d4-aa3f103db97c.log4jdns.northwave.nl was seen
```

[1]: https://github.com/tangxiaofeng7/apache-log4j-poc
[2]: https://logging.apache.org/log4j/2.x/
[3]: https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-44228,==
[4]: https://northwave-security.com/threat-response-remote-code-execution-vulnerability-in-log4j-library/