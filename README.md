# HOWTO Use `pulseaudio` with a Fe-Pi Stereo Sound Card and Fldigi/Direwolf
Version: 20200528  
Author: Steve Magnuson, AG7GN  

This is a variation of the Split Channels documentation for DRAWS/UDRC by **mcdermj** 
at [NW Digital Radio](https://github.com/nwdigitalradio/split-channels), expanded, adapted and modified for use with the Nexus DR-X/DigiLink and Fe-Pi audio.
## 1. Prerequisites

- Raspberry Pi 3B or 3B+.  I have not tested with a model 4B, but it should work.
- [Fe-Pi Audio Z Version 2 sound card](https://fe-pi.com/products/fe-pi-audio-z-v2)
- WB7FHC Budd Churchward's excellent DigiLink (REV C or later) or [Nexus DR-X](http://wb7fhc.com/nexus-dr-x.html) board, or equivalent GPIO controlled PTT
- Raspbian Stretch or Buster OS.  This procedure has not been tested on other distributions, and some instructions are OS-specific.
- OPTIONAL: Speakers attached to Pi's built-in audio jack if you want to monitor the radio's TX and/or RX on the Pi's speakers
- Familiarity with the Pi's Terminal application, basic LINUX commands and the use of `sudo`
- Familiarity with any Linux text editor

This procedure assumes the operating user is __pi__ and pi's home directory is __/home/pi__ and that user __pi__ has sudo privileges (the default).  Adjust as necessary if this is not the case.

## 2. Configuration Procedure

__NOTE:__ If you are using the Hampi image, the majority of the following setup is already done for you.

### 2.0 Modify `/boot/cmdline.txt`

The release of Rasbian Buster stable kernel 4.19.118-v7+ (Raspbian package `raspberrypi-kernel/testing,now 1.20200512-2 armhf`) introduced some changes in how the sound interfaces are handled.  These changes break the Fldigi alerts and TX/RX monitor features.  

Fortunately, the fix is simple: Append this text to the end of the line in `/boot/cmdline.txt` and reboot.  This is ONLY needed when using raspbian-kernel `1.20200512-2` or later.

	snd-bcm2835.enable_compat_alsa=1 snd-bcm2835.enable_hdmi=0 snd-bcm2835.enable_headphones=0
	
You must reboot after making this change.

### 2.1 Install `pulseaudio`  

`pulseaudio` is installed by default in Raspbian Stretch and Buster.  Just to make sure it is, open a terminal, then run:

	sudo apt update && sudo apt install pulseaudio

### 2.2 Determine the sound card name

Open a terminal and run `arecord -l` as shown in this example:

	pi@mypi:~ $ arecord -l 
	card 1: Audio [Fe-Pi Audio], device 0: Fe-Pi HiFi
	sgtl5000-0 [] Subdevices: 0/1 Subdevice #0: subdevice #0

The card name is "__Audio__" in this example.  Use the name that's after "card 1:" and before the text in square brackets.  For Fe-Pi, it'll be "__Audio__"

### 2.3 Back up any existing `/etc/asound.conf` file

- Case 1: There is no existing `/etc/asound.conf` file.  Example:

		pi@mypi:~ $ ls /etc/asound.conf 
		ls: cannot access '/etc/asound.conf': No
		such file or directory

	...no file, so nothing to back up.

- Case 2: There is an existing file.

		pi@mypi:~ $ ls /etc/asound.conf
		/etc/asound.conf

	...back it up.  Run:

		sudo cp /etc/asound.conf /etc/asound.conf.original 

### 2.4 Create/modify `/etc/asound.conf`

This step will make the virtual audio interfaces we create in `pulseaudio` available to applications that can't use `pulseaudio` directly, like Direwolf.  These appear to other applications as [ALSA](https://alsa-project.org/wiki/Main_Page) devices.  

As sudo, edit or create `/etc/asound.conf` in a text editor.  It should contain only the following lines.  

	pcm.fepi-capture-left {
	  type pulse
	  device "fepi-capture-left"
	}
	pcm.fepi-playback-left {
	  type pulse
	  device "fepi-playback-left"
	}
	pcm.fepi-capture-right {
	  type pulse
	  device "fepi-capture-right"
	}
	pcm.fepi-playback-right {
	  type pulse
	  device "fepi-playback-right"
	}
	pcm.system-audio-playback {
	  type pulse
	  device "system-audio-playback"
	}

### 2.5 Modify `/etc/pulse/default.pa`

The changes to this file will set up the Fe-Pi left and right channels and tell `pulseaudio` to use the NULL audio source (no input audio) and the Pi's built-in sound card for the audio sink (output) by default.  

We will use environment variables __PULSE_SOURCE__ and __PULSE_SINK__ when starting Fldigi so that it'll use the Fe-Pi audio interface instead of NULL/built-in-audio-card.  

Likewise, we'll specify the left and right virtual interfaces we create in __default.pa__ below in __direwolf.conf__ to point direwolf to the correct radio by referencing the virtual ALSA devices we created in the previous step.  

Finally, we'll create the virtual interface `system-audio-playback` for use within Fldigi for alerts, notifications, and audio monitoring.

As sudo, make a backup of `/etc/pulse/default.pa`:

	cd /etc/pulse 
	sudo cp default.pa default.pa.original 

As sudo, edit `/etc/pulse/default.pa` so it looks like the `default.pa` file in this repository.  

### 2.6 Prevent `pulseaudio` from being the default sound interface  

We don't want PulseAudio to be the default audio device.  Why?  Because we don't want applications that use audio to interfere with the operation of the ham radio applications.

__IMPORTANT__:  I have noticed that when `pulseaudio` is updated as part of the normal Raspbian updates, `/usr/share/alsa/pulse-alsa.conf` may be overwritten.  Please check this file whenever `pulseaudio` is updated and make sure that all lines are commented out as described below.

Make a copy of `/usr/share/alsa/pulse-alsa.conf`:  

	sudo cp /usr/share/alsa/pulse-alsa.conf /usr/share/alsa/pulse-alsa.conf.original  

As sudo, edit `/usr/share/alsa/pulse-alsa.conf` and comment out __all__ non-empty lines so it looks like this:

	# This file is referred to by /usr/share/alsa/pulse.conf to set pulseaudio as
	# the default output plugin for applications using alsa when PulseAudio is
	# running.

	#pcm.!default {
	#    type pulse
	#    hint {
	#        show on
	#        description "Playback/recording through the PulseAudio sound server"
	#    }
	#}

	#ctl.!default {
	#    type pulse
	#}

### 2.7 Restart `pulseaudio` 

- For Debian Buster, `pulseaudio` runs as a __user__ service:

	Open a terminal, then run:

		systemctl --user restart pulseaudio 
	
- For Debian Stretch, `pulseaudio` runs as a __sytem__ service:

	Open a terminal, then run:

		pulseaudio -k 
		pulseaudio --start --log-target=syslog

## 3. Scenarios

At this stage, `pulseaudio` is ready to use on either the left or the right radio or both at once.   

1. Decide how you want to use your radios and Fldigi and Direwolf.  

1. Determine where your radio(s) is/are connected (applies to the Nexus DR-X and DigiLink boards):

	- The "Left" radio (uses GPIO 12 for PTT) is plugged in to the left 
6-pin miniDIN **_or_** the RJ45 jack if configured.
	- The "Right" radio (uses GPIO 23 for PTT) is plugged in to the right 
6-pin miniDIN **_or_** the TRRS jack if configured.

1. Select your desired scenario from the __Scenario__ sections that follow.

### 3.1 Scenario 1: I want to use only Fldigi with one radio.

#### 3.1.1 Configure Fldigi (4.1.09 and later)

1. Open Fldigi, click __Configure > Config Dialog__.  Select __Soundcard > Devices__.
1. Check __PulseAudio__ and leave the *Server String* field empty. 
1. Select __Settings__ and select Converter __Medium Sinc Interpolator__ (best for V/UHF - change as desired) 
1. Select __Right channel__.  
	For "Right" connected radios: Check _Reverse left/right channels_ under 
__Transmit__ Usage *and* under __Receive__ Usage.  
  For "Left" connected radios, leave these 2 boxes unchecked. 
1. Select __Rig Control > GPIO__ and set your GPIO to whatever your radio uses: 
  	- "Left" radios: __BCM 12__, also check the corresponding '__= 1 (on)__' box  
  	- "Right" radios: __BCM 23__, also check the corresponding '__= 1 (on)__' box
1. Click __Save__ and __Close__.
1. Restart Fldigi.

#### 3.1.2 Locate and edit your `fldigi.desktop` file

1. Open the __/usr/local/share/applications/fldigi.desktop__ file in an editor (as sudo) and change the `Exec=` line as follows, then save the file and exit the editor:

		Exec=sh -c 'PULSE_SINK=fepi-playback PULSE_SOURCE=fepi-capture fldigi'

1. Reload the menu items.  Run this command in a terminal (this step may not be necessary):

		lxpanelctl restart

1.	[Re]start Fldigi from the menu.

### 3.2 Scenario 2: I want to use only Direwolf with one radio.

#### 3.2.1 Configure Direwolf

In your direwolf.conf file (usually located in your home folder
or in `/etc`), change the ADEVICE line to look like this:

	ADEVICE fepi-capture-? fepi-playback-?

where *?* = `left` or `right` depending on where your radio is plugged in to the Nexus DR-X/DigiLink.

Also change the PTT line to 

	PTT GPIO y

where *y* = __12__ for the left radio or __23__ for the right radio.

#### 3.2.2 Restart Direwolf using whatever script you use to operate Direwolf.

### 3.3 Scenario 3: I want to run Fldigi on the {right or left} radio and Direwolf on the {left or right} radio at the same time.  
                
*Note that if Direwolf is on the Left radio, then Fldigi must be on the Right radio or vice-versa.  Fldigi and Direwolf cannot both use the same radio at the same time.*

#### 3.3.1 Follow the directions in Scenarios 1 and 2.

### 3.4 Scenario 4: I want to run 2 Fldigi applications at the same time, one for a "Right" radio and one for a "Left" radio.

__Attention Flmsg users__:  It is possible but somewhat complicated to run 2 instances of Flmsg, one associated with the left Fldigi and one with the right.  I recommend keeping things as simple as possible:  If you want to use Flmsg, make sure only one instance of Fldigi is running.  

The Hampi image comes already set up to do run and Flmsg "Left" and Flmsg "Right" corresponding to Fldigi Left and Right.

#### 3.4.1 Make copies of the Fldigi configuration folder.  

Open a terminal, then run:

	cd ~
	cp -r .fldigi/ .fldigi-left/
	cp -r .fldigi/ .fldigi-right/

#### 3.4.2 Locate your `fldigi.desktop` file.

__SECTION DELETED__

#### 3.4.3 Make new menu entries for each radio.

1) Example: Say your 2 radios are a Kenwood (on the left) and an Alinco (on the
right), then run:

		cd /usr/local/share/applications/
		sudo cp fldigi.desktop fldigi-kenwood.desktop 
		sudo cp fldigi.desktop fldigi-alinco.desktop 
		sudo mv fldigi.desktop fldigi.desktop.disabled

1) Edit (as sudo) the `fldigi-kenwood.desktop` file, changing the `Exec=` and `Name=`
lines as follows:

		Exec=sh -c 'PULSE_SINK=fepi-playback PULSE_SOURCE=fepi-capture fldigi --config-dir=/home/pi/.fldigi-left -title "Kenwood fldigi"' 
		Name[en_US]=Fldigi - Kenwood

1) Edit (as sudo) the `fldigi-alinco.desktop` file, changing the `Exec=` and `Name=`
lines as follows:

		Exec=sh -c 'PULSE_SINK=fepi-playback PULSE_SOURCE=fepi-capture fldigi --config-dir=/home/pi/.fldigi-right -title "Alinco fldigi"' 
		Name[en_US]=Fldigi - Alinco

1) Reload the menu.  Run this command in a terminal (this step may not be necessary):

		lxpanelctl restart

You should now see 2 Fldigi entries in your menu.

1) Run ONE (*only run one at a time during setup!*) of the new Fldigi menu items and set up the sound card and Rig control settings as described in Scenario 1.  Close that instance of Fldigi and repeat for the second Fldigi menu item.
       
1) Now you can run both the Alinco and Kenwood Fldigis at the same time. 

## 4. Adjust audio settings  

### 4.1 Start `alsamixer`
Open a terminal and run:

	alsamixer

Use the arrow keys to select the control.  Notes on using `alsamixer`:

1. The default card reported by `alsamixer` should be the Pi's built-in card (bcm2835 ALSA).   
1. Some controls are in stereo.  The up and down arrows change the levels of both left channels (for the left radio) and right (for the right radio).
      
	Pressing __Q__ or __E__ increases the left or right channel (radio) level respectively. 
	
	Pressing __Z__ or __C__ decreases the left or right channel (radio) level respectively.

1. Press __F6__ and select __1 Fe-Pi Audio__ from the list.   For the Fe-Pi, these levels 
are good starting points:

	Headphone: _00_   
	Headphone Mux: _DAC_   
	Headphone Playback ZC: _00_  
	PCM: _89_ (left), _89_ (right)    
	Lineout: _100_ (left), _100_ (right)   
	Mic: _0_   
	Mic: _0_  
	Capture: _13_ (left), _13_ (right)  
	Capture Attenuate Switch: _00_  
	Capture Mux: _LINE_IN_   
	Capture ZC: _00_   
	AVC: _MM_   
	AVC Hard Limiter: _MM_     

	Leave the remaining settings as-is.  
	
	You will want to open `alsamixer` while running Fldigi and/or direwolf and adjust
these settings on the __1 Fe-Pi Audio__ device as needed: 

	__Capture__ (for audio coming from the radio into the Pi - radio RX)  
	__PCM__ (for audio coming from the Pi to the radio - radio TX)

	Once you're happy with your audio settings, press __Esc__ to exit alsamixer.  

1. Save Audio Settings (usually not required)
	
	These audio settings should save automatically, but for good measure you can store them again by running:

		sudo alsactl store

	If you want to save different audio level settings for different scenarios, 
you can run this command to save the settings (choose your own file name/location):

		sudo alsactl --file ~/mysoundsettings1.state store

	...and this command to restore those settings:

		sudo alsactl --file ~/mysoundsettings1.state restore

## 5. (Optional) Monitor Radio's TX and/or RX audio on Pi's Speakers

If you have speakers connected to the Pi, you can configure `pulseaudio` to monitor the audio to and/or from the radio

The Pi's built-in sound interface can output audio to the audio jack on the board.  This is the __Analog__ output.  It can also send audio to HDMI-attached monitors that have built-in speakers.  This is the __HDMI__ output.  To toggle between __Analog__ and __HDMI__, right-click on the speaker icon on the menu bar. Move your mouse over __Audio Outputs__.  Another menu should pop out showing __Analog__ and __HDMI__.  Select the one appropriate for your setup.

To adjust the level of the audio on your Pi's speakers, use the speaker's volume knob if it has one.  The speaker icon in the upper right of the Pi desktop also controls the Pi's speaker volume.  

Another way is to adjust the volume in alsamixer (__0 bcm2835 ALSA__ device) or clicking the Raspberry icon on the desktop, select __Preferences > Audio Device Settings__.  Select the __bcm2835 ALSA__ sound card, and adjust the slider to your liking.  You may have to click the __Select Controls...__ button to enable the slider.

### 5.1 Control TX and/or RX monitoring control from the command line

These terminal commands will enable and disable monitoring of the radio's TX and RX on the Pi's speakers.  None, one or both of these monitors can be run at the same time.

- TX monitoring  

		pactl load-module module-loopback source="fepi-playback.monitor" sink="system-audio-playback"
- RX monitoring  

		pactl load-module module-loopback source="fepi-capture" sink="system-audio-playback"
- Stop all monitoring

		pactl unload-module module-loopback

Make sure your speakers are attached (Ananlog or HDMI) and your have Analog or HDMI output selected and the volume is set correctly.

### 5.2 (Optional) Make some menu items to control RX and TX monitoring

You can create menu items to control RX and TX monitoring by placing the above commands in a `.desktop` file.
You may need to modify __Categories=__ in the files described below depending on how you have your Raspberry Pi menus set up so that they appear in the correct top level menu.

1.	Create (as sudo) a file called `/usr/local/share/applications/radio-monitor-rx.desktop` with this text:

		[Desktop Entry]
		Name=Start RX Monitor
		GenericName=Amateur Radio Digital Modem
		Comment=Amateur Radio Sound Card Communications
		Exec=sh -c 'pactl load-module module-loopback source="fepi-capture" sink="system-audio-playback"'
		Icon=fldigi
		Terminal=false
		Type=Application
		Categories=HamRadio;
		Path=/home/pi

1.	Create (as sudo) a file called `/usr/local/share/applications/radio-monitor-tx.desktop` with this text:

		[Desktop Entry]
		Name=Start TX Monitor
		GenericName=Amateur Radio Digital Modem
		Comment=Amateur Radio Sound Card Communications
		Exec=sh -c 'pactl load-module module-loopback source="fepi-playback.monitor" sink="system-audio-playback"'
		Icon=fldigi
		Terminal=false
		Type=Application
		Categories=HamRadio;
		Path=/home/pi
		
1.	Create (as sudo) a file called `/usr/local/share/applications/radio-monitor-stop.desktop` with this text:

		[Desktop Entry]
		Name=Stop TX and RX Monitors
		GenericName=Amateur Radio Digital Modem
		Comment=Amateur Radio Sound Card Communications
		Exec=sh -c 'pactl unload-module module-loopback'
		Icon=fldigi
		Terminal=false
		Type=Application
		Categories=HamRadio;
		Path=/home/pi
		
1. Edit the placement of these menu items as desired using the Pi's Main Menu editor (__Pi icon > Preferences > Main Menu Editor__)

### 5.3 (Optional) Using Fldigi 4.1.09 (and later) Alerts, Notifications and RX Monitoring

Fldigi version 4.1.09 introduced the ability to control where alerts, notifications and monitoring audio is sent.  Obviously, you don't want to send that audio to the radio so having the ability to control where it goes is important.

5.3.1 Fldigi Alerts

Note that Fldigi [Alerts](http://www.w1hkj.com/FldigiHelp/audio_alerts_page.html) won't work for FSQ because it only looks in the signal browser for any text you specify in the alert.

The sound interface used for Alerts and RX Monitoring is set in __Configure > Config Dialog > Soundcard > Devices__.  

1.	In the __Alerts/RX Audio__ section, select `system-audio-playback` and check `Enable Audio Alerts`.  Click __Save__ and __Close__.
1.	In the left pane of the __Fldigi configuration__ window, select __Alerts__ to set up your alerts.  

All alerts will now play through the Pi's built-in sound interface.  Don't forget to select __Analog__ or __HDMI__ output as described earlier.

5.3.2 Fldigi Notifications

Unlike Alerts, Fldigi [Notifications](http://www.w1hkj.com/FldigiHelp/notifier_page.html) can look for text in Fldigi's Receive Messages pane.  To set up an audio alert in Fldigi, open __Configure > Notifications__ 

1.	Enter your search criteria.
2.	Under __Run Program__, enter the following:

		aplay -D system-audio-playback <path-to-WAV-file>
	For example:

		aplay -D system-audio-playback /usr/lib/libreoffice/share/gallery/sounds/untie.wav

This will send audio triggered by a Notification to the built-in audio interface.  Don't forget to select __Analog__ or __HDMI__ output as described earlier.

You can also use `paplay`, the PulseAudio player, which can play OGG audio files in addition to WAV files.  You can optionally tell `paplay` what audio sink to use.  Example:

		paplay --device=system-audio-playback /home/pi/fsq_ag7gn.ogg
		
Use this command list the audio formats `paplay` supports:

		paplay --list-file-formats

Both `aplay` and `paplay` are installed by default in Raspbian Buster.

5.3.3 Fldigi RX Monitoring

Fldigi has built-in audio monitoring capability.  You can toggle RX monitoring on and off and apply filtering to the received audio by going __View > Rx Audio Dialog__.  The audio will be played through the built-in audio interface.  Don't forget to select __Analog__ or __HDMI__ output as described earlier.
