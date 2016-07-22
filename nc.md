##netcat commands

####check if a tcp port is open on remote host
`nc -vz -w1 <hostname> 8089`


####check if a udp port is open on remote host
`nc -vz -4u -w1 <hostname> 8089`

####send text to test syslog reception
`echo "this is a test message" | <netcat-command>`

####run in a loop
`while true; do nc -vz -w5 <hostname> 9997; done`

##resources
http://mikeberggren.com/post/53883822425/ncudp

```
-n - Tells the echo command to not output the trailing newline.
-4u Use IPV4 addresses only.  Use UDP instead of TCP.
-w1 Silently close the session after 1 second of idle time.  That way, we’re not stuck waiting for more data.
```
