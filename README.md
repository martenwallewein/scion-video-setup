# scion-video-setup
Video Setup information for "Design and Implementation of DVB-T Streaming over the SCION Network"

## Mumudvb
Start params: `/usr/bin/mumudvb -d -vvv -c /etc/mumudvb/1.conf`

Conf:
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

## Live Demo setup
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

FFmpeg: `/usr/bin/ffmpeg -i http://49.12.6.5:81/bysid/897 -c:v libvpx-vp9 -b:v 12M -cpu-used 4 -threads 8 -vsync cfr -r 20  http://localhost:8090/feed.ffm`
