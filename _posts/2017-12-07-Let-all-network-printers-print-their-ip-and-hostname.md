This quick hack scans for all network printers, looks up their hostnames and tries to print hostname+ip on each of them:

```shell
for i in `nmap -p9100 192.168.0.0/24 --open -oG - | grep "9100/open" | cut -d' ' -f2`; do
  hostname="`host $i 192.168.0.1 | grep name | cut -d' ' -f5`"
  printf "I am $hostname at $i\n" | nc -w2 $i 9100
done
```

This is not very elegant or safe, as it just sends raw text to port 9100 and times out after 2 seconds. Buggy printer firmware (Kyocera!!!) might try to handle the text input as postscript or other language, which produces various errors.  
It may be "interesting" to try what can be done with/to those printers when sending malformed postscript...


_Thanks_  
_sko_

----
