# plantpi
Documenting the project of turning a Raspberry Pi Zero 2W into a time-lapse camera for an indoor garden.

## Quick notes
* The documentation for the camera apps appears to be in a half-updated state.  They say they're full the Bullseye release, but the commands they use `rpicam` are only available starting in the Bookworm release.
* Bullseye is still the default recommended OS for the Zero 2W in the OS imaging tool, but it isn't clear why.  Bookworm does work.
* The imaging tool also suggests a desktop environment for the Zero 2W, but the device really struggles to run a GUI due to limited RAM.
* The resolution to this is to select no filtering in the OS imaging tool, and select the Bookworm Lite image.
* The Zero 2W without a dekstop environment seems to shut down the display just after startup.  This means the camera preview isn't visible.
* The VNC server can be installed on the pi with just a command line interface, but it doesn't appear to work.

## Links currently under investigation

1.  [Remote access on the Pi](https://www.raspberrypi.com/documentation/computers/remote-access.html)
2.  [Camera software documentation](https://www.raspberrypi.com/documentation/computers/camera_software.html)

## Links from Carl K at Python Office Hours
1.  https://github.com/CarlFK/pici/blob/main/ansible/roles/cam/pi/files/gst-libcam.sh#L23
2.  https://github.com/CarlFK/pici/issues/15
3.  https://github.com/pikvm/ustreamer
4.  https://github.com/CarlFK/pici/tree/main/tools/mkpisd
## Strategy
### The zero-code way

The simplest thing to do is use `rpicam-jpeg` or `rpicam-stills` to set up the time lapse, then share the folder with the pictures using SMB or a similar folder sharing technology.  I can set this to start up automatically using chron.  The downside to this approach is that I, or whoever positions the camera, would have to look at the shared folder as new pictures appear to ensure the camera was aimed properly.  It's not convenient to access SMB shares or the like on smartphones, so this isn't very useable while working on the garden.  The upside is that it would produce ressults quickly and allow me to move on to investigating methods of stitching the pictures into video.

### The challenging but useable way

Ideally users would be able to see where the camera was pointed on their phones, and even signal for an autofocus cycle.  This would require a web server.  It's not difficult to set up a time-lapsee using Python and the example code provided.  But, signaling that the preview should be updated and triggering autofocus might require emitting events into the same loop that controls the time-lapse.  This might require asyncio in Python, and some form of inter-process communication.

## Images to video

There appear to be multiple ways to do this.  The easiest to do programmatically or through the command line is probably https://ffmpeg.org/ based on some quick searches and reading forum posts.  But the forum posts were light on details.
