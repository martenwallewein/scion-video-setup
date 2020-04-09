# scion-video-setup
Video Setup information for "Design and Implementation of DVB-T Streaming over the SCION Network".The setup consists of a Raspberry Pi 4, one server running SCION and a client (vlc for direct access, browser for the live demo setup).

The Raspberry PIU has the IP 141.44.25.152 and the server 141.44.25.148

## Installation on the Raspberry PI
At first, install the required packages (Assuming that SCION already runs on your PI):

`sudo apt-get update`
`sudo apt-get install mumudvb w-scan`

## Mumudvb

### Get channels
To get all accessible channels, run `w-scan -cDE >> channels.conf` as example for germany. See documentation of w-scan https://wiki.ubuntuusers.de/w_scan/.

### Configure mumudvb

Write the config file `/etc/mumudvb/1.conf` which has the following content (frequency is again setup for germany):
```
card=0
tuner=0
freq=666000
bandwidth=8MHz
autoconfiguration=full
autoconf_unicast_start_port=32000
unicast=1
delivery_system=DVBT2
multicast=0
port_http=8899
```

And run mumudvb with `/usr/bin/mumudvb -d -vvv -c /etc/mumudvb/1.conf`

Or configure this one as service:
`vi /etc/systemd/system/mumudvb.service`

```
[Unit]
Description=Mumudvb

[Service]
ExecStart=/usr/bin/mumudvb -d -vvv -c /etc/mumudvb/1.conf

[Install]
WantedBy=multi-user.target
```

And start with `sudo systemctl start mumuvdb`

## SCION HTTP Proxy
At first, create valid TLS certs with `openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes`. Assuming you have build the SCION HTTP Proxy (https://github.com/martenwallewein/scionhttpproxy) and copied it on the Raspberry PI. Then start the proxy with

`SCION_CERT_KEY_FILE=/home/pi/mgartner/key.pem SCION_CERT_FILE=/home/pi/mgartner/cert.pem home/pi/mgartner/scionhttpproxy --local="19-ffaa:1:bcc,[141.44.25.152]:9001" --remote="http://141.44.25.152:8899" --direction=fromScion --cert /home/pi/mgartner/cert.pem --key /home/pi/mgartner/key.pem`

Notes:
You have to insert correct IP's and ASes for you setup. Local is the Raspberry PI and remote the server mentioned above. Furthermore, you may adapt the cert paths. 

As a service:
`vi /etc/systemd/system/scionhttp.service`

```
[Unit]
Description=SCIONHTTP

[Service]
ExecStart=/home/pi/mgartner/scionhttpproxy --local="19-ffaa:1:bcc,[141.44.25.152]:9001" --remote="http://141.44.25.152:8899" --direction=fromScion --cert /home/pi/mgartner/cert.pem --key /home/pi/mgartner/key.pem
Environment=SCION_CERT_KEY_FILE=/home/pi/mgartner/key.pem
Environment=SCION_CERT_FILE=/home/pi/mgartner/cert.pem

[Install]
WantedBy=multi-user.target
```

And start with `sudo systemctl start scionhttp`

## SCION HTTP Proxy on server
To run the server counterpart (which can be accessed from outside) run `SC=/etc/scion SCION_CERT_KEY_FILE=key.pem SCION_CERT_FILE=cert.pem ./newscionhttpproxy --remote="19-ffaa:1:bcc,[141.44.25.152]" --local="19-ffaa:1:bcc,[141.44.25.152]" --localurl="141.44.25.151:8088" --direction=toScion`. Consider again changing IP's and cert paths.

## Accessing the stream
The stream can now be accessed using VLC Media Player, open a network stream and paste in http://141.44.25.151:8088/bysid/897, where `897` is a channel id (look at the channels.conf created before).

## Live Demo setup
The live setup contains an additinal ffmpeg component that runs on the server and transcodes the video. 

ffserver: `/usr/bin/ffserver -f /opt/ffmpeg/ffserver.conf`

Conf: 
```
# Port on which the server is listening. You must select a different
# port from your standard HTTP web server if it is running on the same
# computer.
Port 8090

# Address on which the server is bound. Only useful if you have
# several network interfaces.
BindAddress 0.0.0.0

# Number of simultaneous HTTP connections that can be handled. It has
# to be defined *before* the MaxClients parameter, since it defines the
# MaxClients maximum limit.
MaxHTTPConnections 2000

# Number of simultaneous requests that can be handled. Since FFServer
# is very fast, it is more likely that you will want to leave this high
# and use MaxBandwidth, below.
MaxClients 1000

# This the maximum amount of kbit/sec that you are prepared to
# consume when streaming to clients.
MaxBandwidth 10000

# Access log file (uses standard Apache log file format)
# '-' is the standard output.
CustomLog -

##################################################################
# Definition of the live feeds. Each live feed contains one video
# and/or audio sequence coming from an ffmpeg encoder or another
# ffserver. This sequence may be encoded simultaneously with several
# codecs at several resolutions.

<Feed feed.ffm>

# You must use 'ffmpeg' to send a live feed to ffserver. In this
# example, you can type:
#

File /tmp/feed.ffm
FileMaxSize 100M

# You could specify
# ReadOnlyFile /saved/specialvideo.ffm
# This marks the file as readonly and it will not be deleted or updated.

# Only allow connections from localhost to the feed.
ACL allow 127.0.0.1

</Feed>


##################################################################
# Now you can define each stream which will be generated from the
# original audio and video stream. Each format has a filename (here
# 'test1.mpg'). FFServer will send this stream when answering a
# request containing this filename.
<Stream streamwebm>
#in command line run : ffmpeg -i yourvideo.mp4 -c:v libvpx -cpu-used 4 -threads 8    http://localhost:8090/streamwebm.ffm
Feed feed.ffm
Format webm

# Video Settings
VideoFrameRate 20
VideoSize 624x368

# Audio settings
AudioCodec libvorbis
AudioSampleRate 48000
AVOptionAudio flags +global_header

MaxTime 0
AVOptionVideo me_range 16
AVOptionVideo qdiff 4
AVOptionVideo qmin 4
AVOptionVideo qmax 20
#AVOptionVideo good
AVOptionVideo flags +global_header

# Streaming settings
PreRoll 10
StartSendOnKey

Metadata author "author"
Metadata copyright "copyright"
Metadata title "Web app name"
Metadata comment "comment"

</Stream>

```

FFmpeg: `/usr/bin/ffmpeg -i http://141.44.25.151:8088/bysid/897 -c:v libvpx-vp9 -b:v 12M -cpu-used 4 -threads 8 -vsync cfr -r 20  http://localhost:8090/feed.ffm`

Access the strean with `http://141.44.25.151:8090/feed.ffm` in a webbrowser.