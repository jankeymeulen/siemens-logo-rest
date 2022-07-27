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

# Getting a shared secret to authenticate with the LOGO programmatically

This is reverse engineered from the LOGO 8's web server javascript code. It's only tested with a LOGO 8. Important: your language needs to have unsigned int32 or larger. For instance, regular 32bit PHP is a bad choice. PHP 32bit can achieve this with the gmp-extension, but not natively.

The Login procedure has two steps. The first step is creating four random keys, sending them to the LOGO and getting a challenge in response. The second step is to mash up the password with the server challenges and the four keys. The result for the second request will be the Security-Hint that can then be used for all further requests.

Tokens seem to have unlimited lifetime. Maybe they get lost when rebooting the LOGO. The "keep me logged in" checkbox only seems to tell the browser to store the Security-Hint. Checking or Unchecking the box has no effect on the messages sent and received from the LOGO.

1. pick four random numbers between 0 and 2^32-1 (4294967295) and store them in variables `a1`, `a2`, `b1`, `b2`.

2. POST-Request to http://logo-ip/AJAX with Header "Security-Hint: p" and body "UAMCHAL:3,4,`a1`,`a2`,`b1`,`b2`". Example: "UAMCHAL:3,4,1234567890,2345678901,3456789012,456789012"

3. you'll receive a comma-separated string with three sections as reply: status-code, `Login Security-Hint`, and the `serverChallenge`. For example: "700,ABCDEF1234567890ABCDEF1234567890,567890123". Split on "," and check that the split-result has 3 parts, that the first part is int(700) and the third part is also an integer. Parse the `serverChallenge`as uint32 for all further steps.

4. create the PW token base string by concating the password, a + sign, and the serverChallenge: 
`pwToken = 'password' + '+' + serverChallenge`
If the password has non-ansi characters, they need to be UTF8-encoded in the string.

5. truncate the `pwToken` to max 32 chars. Note: this will do nothing unless your password is longer than 21 chars. I have not tested the login process with such a long password.

6. get the CRC32 of the `pwToken`and xor it with the server challenge. This will be the pwToken for the server: `loginPWToken = crc32(pwToken) ^ serverChallenge`
Use the serverChallenge in its uint32 form here.

7. create the loginServerChallenge by xor-ing all four keys and the serverChallenge: `loginServerChallenge = a1 ^ a2 ^ b1 ^ b2 ^ serverChallenge`

8. POST-Request to http://logo-ip/AJAX with Header "Security-Header: `Login Security-Hint`" and body "UAMLOGIN:Web User,`loginPWToken`,`loginServerChallenge`". Example: "UAMLOGIN:Web User,1345678901,1456789012".

Here is a working example in PHP with gmp and curl extension:

```php
$pw = 'password';

$a1 = gmp_random_range(0, "4294967296");
$a2 = gmp_random_range(0, "4294967296");
$b1 = gmp_random_range(0, "4294967296");
$b2 = gmp_random_range(0, "4294967296");

$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, 'http://[logo-ip]/AJAX');
curl_setopt($ch, CURLOPT_HTTPHEADER, array('Security-Hint: p'));
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, "UAMCHAL:3,4,$a1,$a2,$b1,$b2");

$ret = curl_exec($ch); //700,ABCDEF1234567890ABCDEF1234567890,1234567890
$ret = explode(",", $ret);
if (count($ret) == 3 && $ret[0] == "700") {
  $pwToken = substr($pw . '+' . $ret[2], 0, 32);
  $iPWToken = gmp_xor(crc32($pwToken), $ret[2]);
  $iServerChallenge = gmp_xor(gmp_xor(gmp_xor(gmp_xor($a1, $a2), $b1), $b2), $challenge);
  $post = "UAMLOGIN:Web User,$iPWToken,$iServerChallenge";
  curl_setopt($ch, CURLOPT_HTTPHEADER, array('Security-Hint: ' . $ret[1]));
  curl_setopt($ch, CURLOPT_POSTFIELDS, $post);
  $ret = curl_exec($ch); //700,BCDEFA1234567890BCDEF1234567890A
  $ret = explode(",", $ret);
  if (count($ret) == 2 && $ret[0] == "700") {
    //$ret[1] is now the new Security-Hint!
  } else {
    //error during step 2	
  }
}
else {
  //error during step 1
}
```

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

