---
title: "Setting up Twilio SIP Trunks in Asterisk"
date: 2024-12-07T18:24:18-06:00
tags: ["VoIP"]
---
This is a brief guide that explains how to properly setup Twilio's Elastic SIP Trunking service
with an Asterisk server using realtime configuration. This means that Asterisk's
configuration is stored in a SQL server.

# The Big Picture

Before doing any setup, it is crucial to understand how the Elastic SIP Trunk service is to be
used. It is assumed that you are familiar with SIP and VoIP.

Twilio Elastic SIP Trunking lets you make and receive calls to and from the PSTN. This is
done by sending and receiving SIP INVITEs. You can use TLS and SRTP to encrypt calls. The flow
is really just like making any other call to a SIP endpoint.

## Making calls

In order to make calls you need to:

- Send the INVITE from an authorized IP address.
- Use valid credentials (username and password).

The credentials and authorized IP addresses are set on Twilio.

## Receiving calls

In order to receive calls you need to:

- Receive INVITE from Twilio.
- Validate the source IP address belongs to Twilio.

Twilio does not support credentials when sending INVITEs to your server, so the only security
measure available is to validate the IP address. These addresses are available on Twilio's
documentation.

# Outgoing calls

An outgoing call can be placed with the following dial plan functions:


```
exten => _ZZXXXXXXXX,1,Set(CALLERID(all)="Name" <+10987654321>)
    same => n,Set(CHANNEL(accountcode)=${CHANNEL(endpoint)})
    same => n,Dial(PJSIP/${EXTEN}@twilio)
```

It is important to set a proper Caller ID as Twilio will reject calls with invalid ones. The phone
number in the Caller ID must be a registered phone number in Twilio that you can use. These
numbers can be purchased on Twilio or you can verify ownership of an existing one. As for the
name you are free to set whatever you want.

The call is placed by dialing the desired phone (must use E.164) at the Twilio endpoint which
represents the Elastic's SIP Trunk endpoint.

## Setting up the endpoint

Since realtime is being used, the endpoint must be added to the `ps_endpoints` SQL table. We will
also need to set an AOR so Asterisk knows where to send the INVITE and finally a `ps_auths` entry
with the credentials used for authentication with Twilio.

The following SQL queries achieve these goals:

```sql
INSERT INTO ps_auths
    (id, username, password, auth_type)
VALUES
    ('twilio', 'XXXXX', 'XXXXXX', 'userpass');

INSERT INTO ps_aors
    (id, contact, max_contacts, qualify_frequency)
VALUES
    ('twilio', 'sip:user@yourtermination.pstn.twilio.com', 1, 60);

INSERT INTO ps_endpoints
    (id, disallow, allow, outbound_auth, aors, context, transport)
VALUES
    ('twilio', 'all', 'ulaw', 'twilio', 'twilio', 'from-twilio', 'tls');
```

For this example the endpoint has been named `twilio` but you can use any name, just be sure
to update any references to it, for example, in the dial plan. Keep an eye on the context
`from-twilio` as that will be used for inbound calls.

The chosen transport must exist in `pjsip.conf`, it is crucial to assign a transport to the
endpoint, otherwise Asterisk will fail to qualify it.

The termination URL is set in Twilio and must match with the AOR contact. In my case I set it to use
the grable prefix. The username I chose is also grable. Take this into consideration
when reading the SIP traces below.

Qualify frequency specifies how many seconds to wait between each OPTIONS request sent
by Asterisk to qualify Twilio's endpoint.

## Qualification

After creating the endpoint, Asterisk must qualify the endpoint. This sets it as a valid
destination. If the endpoint is not qualified, Asterisk won't place calls to it. To verify
the endpoint has been qualified, you should see an available contact associated to it:

```
asterisk*CLI> pjsip show contacts

  Contact:  <Aor/ContactUri..............................> <Hash....> <Status> <RTT(ms)..>
==========================================================================================

  Contact:  twilio/sip:grable@grable.pstn.twilio.com       6ed81610eb Avail        74.770
```

You can manually qualify the endpoint with the following command. By turning on PJSIP logging,
the SIP messages can be reviewed to verify the signaling process is taking place as expected:

```
asterisk*CLI> pjsip set logger on
PJSIP Logging enabled
asterisk*CLI> pjsip qualify twilio
Qualifying AOR 'twilio' on endpoint 'twilio'
<--- Transmitting SIP request (472 bytes) to TLS:54.172.60.0:5061 --->
OPTIONS sip:grable@grable.pstn.twilio.com SIP/2.0
Via: SIP/2.0/TLS 206.189.160.179:5061;rport;branch=z9hG4bKPj3e1bc5de-c884-4982-a154-687c9499f219;alias
From: <sip:twilio@206.189.160.179>;tag=b067fa7c-5266-4ccf-8adc-8de63e3bef3f
To: <sip:grable@grable.pstn.twilio.com>
Contact: <sip:twilio@206.189.160.179:5061;transport=TLS>
Call-ID: edede3db-57e9-4121-8af7-2d1193a3990b
CSeq: 12910 OPTIONS
Max-Forwards: 70
User-Agent: Asterisk PBX 22.0.0
Content-Length:  0


<--- Received SIP response (428 bytes) from TLS:54.172.60.0:5061 --->
SIP/2.0 200 OK
Via: SIP/2.0/TLS 206.189.160.179:5061;rport=33549;branch=z9hG4bKPj3e1bc5de-c884-4982-a154-687c9499f219;alias;received=206.189.160.179
From: <sip:twilio@206.189.160.179>;tag=b067fa7c-5266-4ccf-8adc-8de63e3bef3f
To: <sip:grable@grable.pstn.twilio.com>;tag=8e428c5f04cc26585c57a1d8b54d92dc.bf4971df
Call-ID: edede3db-57e9-4121-8af7-2d1193a3990b
CSeq: 12910 OPTIONS
Server: Twilio Gateway
Content-Length: 0
```

## Placing a call

Once the endpoint is qualified you can simply place a call. Any problems can usually be debugged
by reviewing the SIP packets being sent or looking into the Twilio dashboard to find
any reported errors. As reference, the following trace was captured from a successfully placed
phone call:

```
INVITE sip:+10987654321@grable.pstn.twilio.com SIP/2.0
Via: SIP/2.0/TLS 206.189.160.179:5061;rport;branch=z9hG4bKPj52eff699-c3fa-4186-b036-44b77ac58373;alias
From: "Zoftko" <sip:+11234567890@206.189.160.179>;tag=c42e26f4-f0f1-4fd0-a17a-39eb79c68b86
To: <sip:+10987654321@grable.pstn.twilio.com>
Contact: <sip:asterisk@206.189.160.179:5061;transport=TLS>
Call-ID: 5a93782f-a93a-4a96-b6bf-2af927d9ab96
CSeq: 30121 INVITE
Allow: OPTIONS, REGISTER, SUBSCRIBE, NOTIFY, PUBLISH, INVITE, ACK, BYE, CANCEL, UPDATE, PRACK, INFO, MESSAGE, REFER
Supported: 100rel, timer, replaces, norefersub, histinfo
Session-Expires: 1800
Min-SE: 90
Max-Forwards: 70
User-Agent: Asterisk PBX 22.0.0
Content-Type: application/sdp
Content-Length:   328

v=0
o=- 1078592751 1078592751 IN IP4 206.189.160.179
s=Asterisk
c=IN IP4 206.189.160.179
t=0 0
m=audio 38568 RTP/SAVP 0 101
a=crypto:1 AES_CM_128_HMAC_SHA1_80 inline:LjwhrgKC5bLthzvgjj1S3NAGjw2pa2lhgAKXF5w3
a=rtpmap:0 PCMU/8000
a=rtpmap:101 telephone-event/8000
a=fmtp:101 0-16
a=ptime:20
a=maxptime:140
a=sendrecv

<--- Received SIP response (439 bytes) from TLS:54.172.60.0:5061 --->
SIP/2.0 100 trying -- your call is important to us
Via: SIP/2.0/TLS 206.189.160.179:5061;rport=33549;branch=z9hG4bKPj52eff699-c3fa-4186-b036-44b77ac58373;alias;received=206.189.160.179
From: "Zoftko" <sip:+11234567890@206.189.160.179>;tag=c42e26f4-f0f1-4fd0-a17a-39eb79c68b86
To: <sip:+10987654321@grable.pstn.twilio.com>
Call-ID: 5a93782f-a93a-4a96-b6bf-2af927d9ab96
CSeq: 30121 INVITE
Server: Twilio Gateway
Content-Length: 0


<--- Received SIP response (676 bytes) from TLS:54.172.60.0:5061 --->
SIP/2.0 407 Proxy Authentication required
CSeq: 30121 INVITE
Call-ID: 5a93782f-a93a-4a96-b6bf-2af927d9ab96
From: "Zoftko" <sip:+11234567890@206.189.160.179>;tag=c42e26f4-f0f1-4fd0-a17a-39eb79c68b86
To: <sip:+10987654321@grable.pstn.twilio.com>;tag=91558776_c3356d0b_18a2dca6-041e-405d-a1a8-b56383c45d90
Via: SIP/2.0/TLS 206.189.160.179:5061;received=206.189.160.179;rport=33549;branch=z9hG4bKPj52eff699-c3fa-4186-b036-44b77ac58373;alias
Server: Twilio
Contact: <sip:172.25.40.62:5060>
Proxy-Authenticate: Digest realm="sip.twilio.com",qop="auth",nonce="dkgRu3NUwBJN5UHD-q-XNwUP0awwrMLV6VIPjHAhaaPx76L3",opaque="79787547fec3e815d5590577bc77769e"
Content-Length: 0


<--- Transmitting SIP request (494 bytes) to TLS:54.172.60.0:5061 --->
ACK sip:+10987654321@grable.pstn.twilio.com SIP/2.0
Via: SIP/2.0/TLS 206.189.160.179:5061;rport;branch=z9hG4bKPj52eff699-c3fa-4186-b036-44b77ac58373;alias
From: "Zoftko" <sip:+11234567890@206.189.160.179>;tag=c42e26f4-f0f1-4fd0-a17a-39eb79c68b86
To: <sip:+10987654321@grable.pstn.twilio.com>;tag=91558776_c3356d0b_18a2dca6-041e-405d-a1a8-b56383c45d90
Call-ID: 5a93782f-a93a-4a96-b6bf-2af927d9ab96
CSeq: 30121 ACK
Max-Forwards: 70
User-Agent: Asterisk PBX 22.0.0
Content-Length:  0


<--- Transmitting SIP request (1404 bytes) to TLS:54.172.60.0:5061 --->
INVITE sip:+10987654321@grable.pstn.twilio.com SIP/2.0
Via: SIP/2.0/TLS 206.189.160.179:5061;rport;branch=z9hG4bKPjd7d2aa19-62fc-4d8f-a0cd-aa1c84b305a3;alias
From: "Zoftko" <sip:+11234567890@206.189.160.179>;tag=c42e26f4-f0f1-4fd0-a17a-39eb79c68b86
To: <sip:+10987654321@grable.pstn.twilio.com>
Contact: <sip:asterisk@206.189.160.179:5061;transport=TLS>
Call-ID: 5a93782f-a93a-4a96-b6bf-2af927d9ab96
CSeq: 30122 INVITE
Allow: OPTIONS, REGISTER, SUBSCRIBE, NOTIFY, PUBLISH, INVITE, ACK, BYE, CANCEL, UPDATE, PRACK, INFO, MESSAGE, REFER
Supported: 100rel, timer, replaces, norefersub, histinfo
Session-Expires: 1800
Min-SE: 90
Max-Forwards: 70
User-Agent: Asterisk PBX 22.0.0
Proxy-Authorization: Digest username="grable", realm="sip.twilio.com", nonce="dkgRu3NUwBJN5UHD-q-XNwUP0awwrMLV6VIPjHAhaaPx76L3", uri="sip:+10987654321@grable.pstn.twilio.com", response="fc7eeb1f837a4569e02bfa616be265c2", cnonce="c72ad22495784a21ba43e5aeb59faca1", opaque="79787547fec3e815d5590577bc77769e", qop=auth, nc=00000001
Content-Type: application/sdp
Content-Length:   328

v=0
o=- 1078592751 1078592751 IN IP4 206.189.160.179
s=Asterisk
c=IN IP4 206.189.160.179
t=0 0
m=audio 38568 RTP/SAVP 0 101
a=crypto:1 AES_CM_128_HMAC_SHA1_80 inline:LjwhrgKC5bLthzvgjj1S3NAGjw2pa2lhgAKXF5w3
a=rtpmap:0 PCMU/8000
a=rtpmap:101 telephone-event/8000
a=fmtp:101 0-16
a=ptime:20
a=maxptime:140
a=sendrecv

<--- Received SIP response (439 bytes) from TLS:54.172.60.0:5061 --->
SIP/2.0 100 trying -- your call is important to us
Via: SIP/2.0/TLS 206.189.160.179:5061;rport=33549;branch=z9hG4bKPjd7d2aa19-62fc-4d8f-a0cd-aa1c84b305a3;alias;received=206.189.160.179
From: "Zoftko" <sip:+11234567890@206.189.160.179>;tag=c42e26f4-f0f1-4fd0-a17a-39eb79c68b86
To: <sip:+10987654321@grable.pstn.twilio.com>
Call-ID: 5a93782f-a93a-4a96-b6bf-2af927d9ab96
CSeq: 30122 INVITE
Server: Twilio Gateway
Content-Length: 0
```

Also, please don't try to hack me. All resources used for this article have been nuked and no
longer exist.

# Receiving a call

After setting your Asterisk's address as the destination for incoming calls on Twilio's website,
you should start receiving SIP INVITEs for any calls that come in, however you still need to:

1. Validate the call actually comes from Twilio.
2. Execute dial plan logic to handle it.

The source IP can be used to validate the call actually comes from Twilio. This also makes
it possible to assign said call to a given endpoint. Once the call is assigned to an endpoint,
execution will start in the specified endpoint's context. This is where the `from-twilio` context
comes in.

## Identifying Twilio

To identify an IP address as Twilio, add the appropiate entry to `ps_endpoint_id_ips`:

```
INSERT INTO
	ps_endpoint_id_ips (id, endpoint, match)
VALUES
	('twilio-oregon', 'twilio', '54.244.51.0/30')
```

After it, any incoming calls should being execution in the context associated with the `twilio`
endpoint:

```
[from-twilio]
exten => +11234567890,1,Goto(attendant,start,1)
```

And again, as reference, here is a SIP capture for an incoming call that was successfully
received:

```
<--- Received SIP request (1455 bytes) from TLS:54.172.60.0:58540 --->
INVITE sip:+11234567890@grable.voip.zoftko.com;transport=tls SIP/2.0
Record-Route: <sip:54.172.60.0:5061;transport=tls;r2=on;lr>
Record-Route: <sip:54.172.60.0;r2=on;lr>
From: <sip:+10987654321@grable.pstn.twilio.com:5060>;tag=08601002_c3356d0b_658bbbdc-370e-4c02-862b-9f5807568f80
To: <sip:+11234567890@grable.voip.zoftko.com;transport=tls>
CSeq: 29044 INVITE
Max-Forwards: 60
P-Asserted-Identity: <sip:+10987654321@184.150.215.12:5060>
Diversion: <sip:+11234567890@twilio.com>;reason=unconditional
Call-ID: fb3e6e86a53e3094ed17b4776a69af59@0.0.0.0
Via: SIP/2.0/TLS 54.172.60.0:5061;branch=z9hG4bK1dc3.f822d4f6a9d2c3b351d555992ddc6b78.0
Via: SIP/2.0/UDP 172.25.75.240:5060;rport=5060;branch=z9hG4bK658bbbdc-370e-4c02-862b-9f5807568f80_c3356d0b_588-14959261184934463487
Contact: <sip:+10987654321@172.25.75.240:5060;transport=udp>
Allow: ACK,BYE,CANCEL,INVITE,OPTIONS,REFER,NOTIFY
X-Twilio-AccountSid: im-not-telling-you
User-Agent: Twilio Gateway
Content-Type: application/sdp
X-Twilio-CallSid: somesid
Content-Length: 364

v=0
o=root 1311022221 1311022221 IN IP4 172.18.165.214
s=Twilio Media Gateway
c=IN IP4 168.86.138.121
t=0 0
m=audio 12692 RTP/SAVP 0 8 101
a=crypto:1 AES_CM_128_HMAC_SHA1_80 inline:h77xBx3cFE4uhlQtSlBa4a8veOdxnbd6OBxy2bAy
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:101 telephone-event/8000
a=fmtp:101 0-16
a=ptime:20
a=maxptime:20
a=sendrecv

<--- Transmitting SIP response (656 bytes) to TLS:54.172.60.0:58540 --->
SIP/2.0 100 Trying
Via: SIP/2.0/TLS 54.172.60.0:5061;rport=58540;received=54.172.60.0;branch=z9hG4bK1dc3.f822d4f6a9d2c3b351d555992ddc6b78.0
Via: SIP/2.0/UDP 172.25.75.240:5060;rport=5060;branch=z9hG4bK658bbbdc-370e-4c02-862b-9f5807568f80_c3356d0b_588-14959261184934463487
Record-Route: <sip:54.172.60.0:5061;transport=tls;lr;r2=on>
Record-Route: <sip:54.172.60.0;lr;r2=on>
Call-ID: fb3e6e86a53e3094ed17b4776a69af59@0.0.0.0
From: <sip:+10987654321@grable.pstn.twilio.com>;tag=08601002_c3356d0b_658bbbdc-370e-4c02-862b-9f5807568f80
To: <sip:+11234567890@grable.voip.zoftko.com>
CSeq: 29044 INVITE
Server: Asterisk PBX 22.0.0
Content-Length:  0

<--- Transmitting SIP response (1271 bytes) to TLS:54.172.60.0:58540 --->
SIP/2.0 200 OK
Via: SIP/2.0/TLS 54.172.60.0:5061;rport=58540;received=54.172.60.0;branch=z9hG4bK1dc3.f822d4f6a9d2c3b351d555992ddc6b78.0
Via: SIP/2.0/UDP 172.25.75.240:5060;rport=5060;branch=z9hG4bK658bbbdc-370e-4c02-862b-9f5807568f80_c3356d0b_588-14959261184934463487
Record-Route: <sip:54.172.60.0:5061;transport=tls;lr;r2=on>
Record-Route: <sip:54.172.60.0;lr;r2=on>
Call-ID: fb3e6e86a53e3094ed17b4776a69af59@0.0.0.0
From: <sip:+10987654321@grable.pstn.twilio.com>;tag=08601002_c3356d0b_658bbbdc-370e-4c02-862b-9f5807568f80
To: <sip:+11234567890@grable.voip.zoftko.com>;tag=aed3c9da-d9a6-441f-8847-f91390679afc
CSeq: 29044 INVITE
Server: Asterisk PBX 22.0.0
Contact: <sip:206.189.160.179:5061;transport=TLS>
Allow: OPTIONS, REGISTER, SUBSCRIBE, NOTIFY, PUBLISH, INVITE, ACK, BYE, CANCEL, UPDATE, PRACK, INFO, MESSAGE, REFER
Supported: 100rel, timer, replaces, norefersub
Content-Type: application/sdp
Content-Length:   328

v=0
o=- 1311022221 1311022223 IN IP4 206.189.160.179
s=Asterisk
c=IN IP4 206.189.160.179
t=0 0
m=audio 32488 RTP/SAVP 0 101
a=crypto:1 AES_CM_128_HMAC_SHA1_80 inline:Nd71WtTEKmtj/nMH2mb9UaG3W+wQKzqOw8uhGTST
a=rtpmap:0 PCMU/8000
a=rtpmap:101 telephone-event/8000
a=fmtp:101 0-16
a=ptime:20
a=maxptime:140
a=sendrecv

<--- Received SIP request (694 bytes) from TLS:54.172.60.0:58540 --->
ACK sip:206.189.160.179:5061;transport=TLS SIP/2.0
Call-ID: fb3e6e86a53e3094ed17b4776a69af59@0.0.0.0
CSeq: 29044 ACK
From: <sip:+10987654321@grable.pstn.twilio.com:5060>;tag=08601002_c3356d0b_658bbbdc-370e-4c02-862b-9f5807568f80
To: <sip:+11234567890@grable.voip.zoftko.com;transport=tls>;tag=aed3c9da-d9a6-441f-8847-f91390679afc
Max-Forwards: 69
User-Agent: Twilio
X-Twilio-CallSid: somesid
Via: SIP/2.0/TLS 54.172.60.0:5061;branch=z9hG4bK1dc3.4d70599707d7d306f72df81cc15f4397.0
Via: SIP/2.0/UDP 172.25.75.240:5060;rport=5060;received=54.167.225.116;branch=z9hG4bK658bbbdc-370e-4c02-862b-9f5807568f80_c3356d0b_583-5224255848525967737
Content-Length: 0
```

# Conclusions

Twilio's Elastic SIP Trunk is a quick way to easily get a phone number and enable PSTN communication
for your Asterisk server. The official configuration guide only shows how to setup the service
with configuration files. This presents a problem for setups using Asterisk's realtime capabilities
as having to modify configuration files is one of the main motives for using a SQL database
instead.

This article has shown it is possible to enable communication with Twilio's service
using realtime configuration too. TLS and SRTP were enabled both on Asterisk and Twilio
to demonstrate secure calling can be performed, however UDP or TCP may be used instead as well.
