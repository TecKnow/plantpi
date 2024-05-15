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

## Scripts for the easy way
### Client script

The bash script below fetches all the images and then uses `ffmpeg` to creatt the timelapse video.  Using a script reduces the chances that someone - by which I mean myself - will accideentally swap the arguments to `scp` or become too frustrated trying to remember the arguments to `ffmpeg`.

```bash
#!/bin/bash
rsync -aPz plantpi.local:timelapse timelapse
ffmpeg -f image2 -pattern_type glob -i 'timelapse/*.jpg' timelapse.mp4
````

### Camera-side script

```bash                                                #!/bin/bash
rpicam-still --timelapse 30000 --timeout 0 --datetime --nopreview --vflip -o /home/perkinsd/timelapse/
```
The above script will take a picture every 30 seconds indefinitely.  The filename will be the datetetime as a series of digits in the format MMDDHHMMSS.jpg and the `--vflip` argument is needed to treat the side of the camera that receives the cable as the top.  the `--nopreview` option cuts down on log output significantly, and doesn't reduce functionality since there's no monitor to view the preview on.

### crontab entry for the camera script

```
@reboot /home/perkinsd/lapse.sh >> /home/perkinsd/lapse_log 2>&1
```
The above script starts the timelapse camera on system reboot.  The log is written to a file because by default, cron would rather mail it and there's no mail agent to do so, so the output is lost. 

# Pi Zero transfer speeds
Copying the images to my desktop to stitch them into a video is very, very slow.  It takes over two hours to transfer about 4.5 gigabytes of data.  This seems unreasonably slow so it should be possible to improve it.  I'm trying tdifferent methods of copying the files from the pi to my desktop, documented below.

It's worth nothing that the pi currently has a 256 GB A2 class microSD card inserted.  I recently learned that A2 cards must be paired with an A2 reader to deliver the required performance.  If an A2 card is paired with an A1 reader, it may perform below the A1 standcard.  This could be a factor, so I might want to repeat this test with an A1 card.  If I did repeat the tests, they'd be on a different pi zero in a different location, although on the same network.  The comparison would not be exact but the results could still be very clear given the lengths of time involved.

The tests were run on a directory containing 5,621 jpeg files occupying 4781721400 bytes or about 4.5 gigabytes.
                                                    
The bash time command outputs its results to stderr, and writing the command to redirect the output of time, not just the command being timed, was a bit fiddly.

The commands used were as follows.

1. { time rsync -aPz plantpi.local:timelapse/ timelapse ; } 2> rsync_time.txt 
2. { time scp -r plantpi.local:timelapse/ timelapse ; } 2> scp_time.txt
3. { time ssh plantpi.local "tar czf - timelapse" | tar xvzf - ; } 2>tar_pipe.txt 

## In the garden
|method|real|user|sys|
|---|---|---|---|
|rsync|173m 12.319s| 0m 21.341s | 0m 31.013s|
|scp | 130m 9.287s | 0m 16.195s | 0m 30.163s |
|tar pipe |97m 56.873s | 0m 33.002s|0m 34.254s |


## At my desk
|method|real|user|sys|
|---|---|---|---|
|rsync| 127m 4.570s| 0m 16.424s | 0m 23.911s |
|scp | 133m 13.183s | 0m 16.920s | 0m 32.234s |
|tar pipe | 107m 43.583s | 0m 36.185s | 0m 36.615 |

## At my desk with an A1 card
|method|real|user|sys|
|---|---|---|---|
|rsync| 36m 54.465s | 0m 12.392s | 0m 20.330s |
|scp | 74m 9.814s | 0m 13.018s | 0m 25.453s |
|tar pipe | 55m 43.859s | 0m 29.817s | 0m 28.856s |

#  Tempreature problems?

The pi has been freezing when I try to retrieve the photos.  The system journal indicated that the WiFi module was the problem.  Some searching suggested that people have fewer problems with this using the 32-bit version of the OS, and that it may be an overheating problem.  I monitored the system temperature during one attempted retrieval and it exceeded 75C, which is apparently the temperature when the Pi should start throttling itself, and it froze instead.

Here's the command I used to monitor the temperature:

```
watch -n 1 vcgencmd measure_temp
```
from https://pimylifeup.com/raspberry-pi-monitor-temperature/

##  Temperature under the 32-bit OS

I reimaged the Pi to use the 32-bit version of the OS.  Now I can transfer the files without overheating if I do it at my desk whereas before it would crash even then.  Unfortunately, it still crashes if I try to download the pictures while it's in the garden.  I've ordered case with fans.  This may cause the camera to vibrate, if we continue wit hthe camera taped to the case.

#  Camera startup
When I switched two the 32-bit version of the OS I started receiving errors from the cron job indicating that no cameras were available.  But running the script manually still worked.  It apepars that the camera isn't ready by the time the cron job runs.  I have introduced a 30 second sleep in the cron job script to get around this.  If I were more ambitious, I'd move the repitition into the schell script instead of relying on the camera command line so that it would keep trying instead of aborting at the first failure.
