# siemens-logo-rest
Trying to make sense of the Siemens LOGO sort-of built-in REST API. Maybe, someday, I'll write a wrapper for it.

# Getting a shared secret to authenticate with the LOGO

1. Enable the web server on the LOGO through LOGOComfort. Put a password there, I wasn't able to get it working without.

2. Browse to your LOGO (go to http://[LOGO-IP] in your browser) and log in, check the *keep me logged on* checkbox.

3. In the side menu, go to *LOGO! Variable*

4. Add a variable, doesn't matter what, could be an input or something else.

5. Open the browser Developer Tools, in Google Chrome this is done with ctrl-shift-c or cmd-shift-c

6. Go to the *network* tab in the Developer Tools.

7. You should see a bunch of requests to *AJAX* being made all the time. Click one of them.

8. Click on *headers*.

9. Scroll down until you see the *Security-Hint* header. Copy the value there.

10. That's it, with this value you should be able to make all the request you want described below. Only works from the same IP address as the one where you used the browser from though. 

# Doing a request

## Reading a variable

To read a variable from your LOGO, do a request to http://[LOGO-IP]/AJAX.

Set a header *Security-Hint* with the token obtained earlier.

Payload for the request is formed like `GETVARS:v1,$TYPE,0,$ADDRESS,1,1`

$TYPE is the kind of variable you want to read, see below. $ADDRESS is the sequence number of the variable, starting from 0.

The response is in the format of `<rs><r i='v0' e='0' v='00'/></rs>`, where the value after `v=` is what you want.


## Setting a variable

To set a variable on your LOGO, do a request to to http://[LOGO-IP]/AJAX, just like reading.

Set a header *Security-Hint* with the token obtained earlier.

Payload from the request is formed like `SETVARS:v0,$TYPE,0,$ADDRESS,1,1,$VALUE`

$TYPE is the kind of variable you want to write, see below. $ADDRESS is the sequence number of the variable, starting from 0.

$VALUE is the value you want to set. 00 for OFF, 01 for ON. Since I don't have any analog ports in use, I haven't tried anything else (yet).

## Variable types

- 132: VM
- 129: I
- 16: NetI
- 130: Q
- 17: NetQ
- 131: M
- 18: AI
- 21: NetAI
- 19: AQ
- 22: NetAQ
- 20: AM
- 12: CURS KEY
- 13: FUNC KEY
- 14: SHIFT REG

## Examples

- Read the status from I1:
`curl 'http://192.168.1.100/AJAX'  -H 'Security-Hint: ABCDEFGHIJKLMNOPQRSTUVWXYZ123456' --data-binary 'GETVARS:v1,129,0,0,1,1'`

- Read the status from Q5:
`curl 'http://192.168.1.100/AJAX'  -H 'Security-Hint: ABCDEFGHIJKLMNOPQRSTUVWXYZ123456' --data-binary 'GETVARS:v1,130,0,4,1,1`

- Set the status on M17 to OFF
`curl 'http://192.168.1.100/AJAX'  -H 'Security-Hint: ABCDEFGHIJKLMNOPQRSTUVWXYZ123456' --data-binary 'SETVARS:v0,131,0,16,1,1,00'`

