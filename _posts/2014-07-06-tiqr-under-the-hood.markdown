# How does tiqr work under the hood?

tiqr uses the OATH Challenge-Response Algorithm (OCRA) as defined in 
[RFC 6287](http://tools.ietf.org/html/rfc6287)

The OCRA algorithm is based on HMAC, as defined in [RFC 2104](https://tools.ietf.org/html/rfc2104).
HMAC creates a Message Authentication Code (MAC) from a cryptographic hash function.
If the hash function is SHA-1, for instance, the resulting MAC is called HMAC-SHA1.
The resulting MAC function takes a secret key and a message as parameters, and returns a fixed-length code (e.g. 160 bits if we choose SHA-1 as the hash function).
The idea is that without knowing the secret key, it is impossible to predict what the output will be for a given message.

OCRA is part of a family of algorithms for generating One Time Passwords (OTPs):
the other two are called HOTP (defined in RFC4226), and TOTP (defined in RFC6238).
All three are based on HMAC codes, but differ in the what messages they use as input to the HMAC function.
For HOTP, the message is simply a counter, TOTP uses a timestamp (e.g. the current time in milliseconds since 1970, modulo 30 seconds), while OCRA uses a random nonce.
Computing an OTP from a nonce and a secret key can be seen as computing a response from a (random) challenge, OCRA is called a Challenge/Response algorithm.

OCRA is designed to be implemented in hardware tokens with primitive user interfaces, e.g. a small keyboard and display like in Vasco's Digipass 260:

![Vasco Digipass 260](http://www.vasco.com/Images/DP260.jpg "Vasco Digipass 260")

Because entering large numbers by humans is error-prone, the HMAC output is truncated to a shorter code.
The exact way this is done is configurable, and will depend on the desired security (shorter codes are less secure).
To specify exactly what the challenge looks like and how the response is computed from it, the OCRA standard defines the OcraSuite - a string containing all the parameters required by the OCRA algorithm.
An example of such a OcraSuite is the string `OCRA-1:HOTP-SHA1-6:QN08`.
This string decomposes into three parts, separated by a colon.
The first (OCRA-1 in this example) specifies the version of the OCRA algorithm.
The current version is 1, but maybe a new version will emerge in the future.
The second part (here, HOTP-SHA1-6) specifies the cryptographic function used when computing the response.
In the example, the basic HOTP algorithm is used (except that we won't be using a counter here), with SHA-1 as the hash function and truncating the response to 6 digits (again, using the truncation algorithm defined in RFC4226).
The final part specifies what the challenge looks like.
QN08 means that the challenge ("question") is numeric (i.e. contains only decimal digits) and has a length of 8 characters.
The OCRA standard defines many more complicated suites, supporting mutual authentication and signatures and incorporating timestamps and counters - but as these features are not used in tiqr, I won't dive into them here.

Instead, let's look at an example.
The ORCA RFC contains some test vectors.
For example: using the suite `OCRA-1:HOTP-SHA1-6:QN08` with the 20 byte key `12345678901234567890` will transform the 8 byte numeric challenge `00000000` into the response `237653`.

How is this response computed?

First, the input to the HMAC function is constructed from several parts (we are ignoring the parts that are not used in this suite).
We first take the OcraSuite string itself, terminated by a 0-character.
Then we concatenate the challenge, padded with 0-characters to a total of 128 bytes.
This yields the message that we will use as input to the HMAC function.
From a UNIX-like shell, we could construct this value for the test vector above, as follows:

	$ (echo -n OCRA-1:HOTP-SHA1-6:QN08; echo -ne '\0'; for i in {1..128}; do echo -ne "\0"; done) | openssl sha1 -hmac "12345678901234567890" -c
	(stdin)= 34:1b:ce:d5:b6:aa:2e:b0:9f:34:d9:3a:06:3c:b5:77:f0:5e:b1:10

Now, we need to truncate this 160-bit value to a 6 digit number.
This is done by first selecting four consecutive bytes from the HMAC output to produce a 32-bit number.
The first of these four bytes is selected by the last nibble of the HMAC output.
In this example, the last hex digit is 0, meaning the first 4 bytes are used: 341bcdd5.
The most significant bit of this 32-bit number is ignored (not applicable here), so we arrive at

	$ printf "%d\n" 0x341bced5 
	874237653

We are only interested in the last 6 digits, however, so the generated OTP is:

	$ printf "%d" 0x341bced5 | tail -c6
	237653

## Enrolment

So, how does the tiqr application use OCRA to authenticate to a web site? Well, before we can authenticate, we need to share a secret with the tiqr Authentication Server.
This process is called enrolment and the exact process will depend on security policies in place.
For the demo tiqr application, there is a simple enrolment page, one that isn't very realistic because there is no way to identify the person that is enrolling.
It is a pure self-service enrolment interface.
However, it is perfect for explaining how enrolment is performed, *technically*.

So, let's browse to (https://demo.tiqr.org/) and browse to the online banking pages to log in.
An authentication page will appear, but as we are not enrolled yet,  we click the "Not yet enrolled" link to set up a new account.

We enter a user name (e.g. johnny) and our full name (John Appleseed) and click "go".
What appears now is a QR tag that we need to scan using the tiqr app.

![tiqr enrolment QR code][enrol]

[enrol]: http://chart.apis.google.com/chart?cht=qr&chs=300x300&chl=tiqrenroll%3A//https%3A//demo.tiqr.org/simplesaml/module.php/authTiqr/metadata.php%3Fkey%3D9902b90015149db00b5eaf3abdc50562%0D%0A&chld=H|0 "tiqr enrolment QR code"


The QR code is used to easily transfer information to our phone.
In fact, the QR encodes the following information:

	tiqrenroll://https://demo.tiqr.org/simplesaml/module.php/authTiqr/metadata.php?key=9902b90015149db00b5eaf3abdc50562

The idea is that the tiqr app can retrieve the embedded URL to obtain metadata for the newly created account.
If we download the metadata:

	$ curl https://demo.tiqr.org/simplesaml/module.php/authTiqr/metadata.php?key=9902b90015149db00b5eaf3abdc50562
	{"service":{"displayName":"tiqr demo","identifier":"demo.tiqr.org","logoUrl":"https:\/\/demo.tiqr.org\/img\/tiqrRGB.png","infoUrl":"https:\/\/www.tiqr.org","authenticationUrl":"https:\/\/demo.tiqr.org\/simplesaml\/module.php\/authTiqr\/post.php","ocraSuite":"OCRA-1:HOTP-SHA1-6:QH10-S","enrollmentUrl":"https:\/\/demo.tiqr.org\/simplesaml\/module.php\/authTiqr\/enroll.php?key=daffc0647da2d5d42449f8f4241542cf"},"identity":{"identifier":"johnny","displayName":"John"}}

The metadata is JSON-encoded.
If we pretty-print this information, we get:

	{ service: 
	   { displayName: 'tiqr demo',
	     identifier: 'demo.tiqr.org',
	     logoUrl: 'https://demo.tiqr.org/img/tiqrRGB.png',
	     infoUrl: 'https://www.tiqr.org',
	     authenticationUrl: 'https://demo.tiqr.org/simplesaml/module.php/authTiqr/post.php',
	     ocraSuite: 'OCRA-1:HOTP-SHA1-6:QH10-S',
	     enrollmentUrl: 'https://demo.tiqr.org/simplesaml/module.php/authTiqr/enroll.php?key=daffc0647da2d5d42449f8f4241542cf' },
	  identity: { identifier: "johnny", displayName: "John" } }

The metadata contains useful information, like the name of the service we're registering at, the ocraSuite that the server wants to use, and most importantly:
the endpoint URLs where we would need to enrol, and where we need to send responses to when authenticating.

The next step is where we exchange a secret.
The tiqr application generated a random secret and shares it with the server.
If is important that the secret is not shared with anyone else, so HTTPS is always used in enrolment URLs.

Let's say we generate the 256-bit secret 0x3132333435363738393031323334353637383930313233343536373839303132 (which doesn't look very random of course, but I think I can get away with it in an example).
The tiqr app will POST the secret to the enrolment URL first:

	curl --data secret=3132333435363738393031323334353637383930313233343536373839303132 https://demo.tiqr.org/simplesaml/module.php/authTiqr/enroll.php?key=daffc0647da2d5d42449f8f4241542cf
	OK

Great! We are enrolled.
Now let's move on to authenticate.

## Authentication

So, let's again browse to (https://demo.tiqr.org/) and navigate to the online banking pages to log in.
An authentication page will appear, and this time we are already enrolled,  so we use our tiqr client to scan the QR code

![tiqr authentication QR code][tiqrauth]

[tiqrauth]: http://chart.apis.google.com/chart?cht=qr&chs=300x300&chl=tiqrauth%3A//demo.tiqr.org/f2fadeb54690d0d71924236f87e090bb/8ab9d15047/demo.tiqr.org&chld=H|0 "tiqr authentication QR code"
http://chart.apis.google.com/chart?cht=qr&chs=300x300&chl=tiqrauth%3A//demo.tiqr.org/f2fadeb54690d0d71924236f87e090bb/8ab9d15047/demo.tiqr.org&chld=H|0

This QR code decodes as:

	tiqrauth://demo.tiqr.org/f2fadeb54690d0d71924236f87e090bb/8ab9d15047/demo.tiqr.org

Embedded in this code are a challenge question (8ab9d15047) and a session key (f2fadeb54690d0d71924236f87e090bb)

If we use the OCRA algorithm to compute the response using the default OCRASuite of `OCRA-1:HOTP-SHA1-6:QH10-S` and above challenge question and session we arrive at the response: 880407.

The tiqr app will POST this response to the authentication URL that was retrieved from the metadata during enrollment.

	$ curl --data sessionKey=f2fadeb54690d0d71924236f87e090bb --data userId=johnny --data response=880407 https://demo.tiqr.org/simplesaml/module.php/authTiqr/post.php
	OK

Hurray! We just succesfully authenticated as user "johnny".
The browser window should now have automatically redirected to a welcome page.
