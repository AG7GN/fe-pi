# HOWTO Use PulseAudio with a Fe-Pi Stereo Sound Card and Fldigi/Direwolf
Version: 20190524  
Author: Steve Magnuson, AG7GN  

This is a variation of the Split Channels documentation for DRAWS/UDRC by **mcdermj** 
at [NW Digital Radio](https://github.com/nwdigitalradio/split-channels).
## 1. Prerequisites

- Raspberry Pi 3B or 3B+
- [Fe-Pi Audio Z Version 2 sound card](https://fe-pi.com/products/fe-pi-audio-z-v2)
- WB7FHC Budd Churchward's excellent DigiLink board (REV C or later) or equivalent GPIO controlled PTT
- Raspbian Stretch OS  (This procedure has not been tested on other distributions)
- OPTIONAL: Speakers attached to Pi's built-in audio jack if you want to replicate radio TX and/or RX to the Pi's speakers. 
- Familiarity with the Pi's Terminal application, basic LINUX commands and the use of sudo
- Familiarity with any Linux text editor

This procedure assumes the operating user is __pi__ and pi's home directory is __/home/pi__.  Adjust as necessary if this is not the case.

## 2. Configuration Procedure

### 2.1 Install pulseaudio.  
Open a terminal, then run:

	sudo apt update && sudo apt install pulseaudio

### 2.2 Select Pi's local audio output destination (speakers or HDMI)

*Skip this step if you have no speakers connected to your Pi.*  

Force the Pi to use either the HDMI connection or the built-in 3.5mm connection for system sounds and for monitoring Fldigi transmissions.  

Open a terminal and run:

	sudo raspi-config

Then:

- Select __7 Advanced Options__ \<ENTER\>
 
- Select __A4 Audio__ \<ENTER\>

	- Select __1 Force 3.5mm (headphone) jack__ if your speakers areconnected there. (Most Common)  
**_OR_**  
	- select __2 Force HDMI__ if your monitor has built-in speakers and can receive audio via HDMI. 
Press \<ENTER\> 

- Select __FINISH__ (Press the TAB key to move the cursor)

### 2.3 Determine the sound card name.

Open a terminal and execute 'arecord -l' as shown in this example:

	pi@mypi:~ $ arecord -l 
	card 1: Audio [Fe-Pi Audio], device 0: Fe-Pi HiFi
	sgtl5000-0 [] Subdevices: 0/1 Subdevice #0: subdevice #0

The card name is "Audio" in this example.  Use the name that's after "card 1:" and before the text 
in square brackets.  For Fe-Pi, it'll be "Audio"

### 2.4 Back up any existing /etc/asound.conf file.

- Case 1: There is no existing /etc/asound.conf file.  Example:

		pi@mypi:~ $ ls /etc/asound.conf 
		ls: cannot access '/etc/asound.conf': No
		such file or directory

	...no file, so nothing to back up.

- Case 2: There is an existing file.

		pi@mypi:~ $ ls /etc/asound.conf
		/etc/asound.conf

	...back it up:

		pi@mypi:~ $ sudo cp /etc/asound.conf /etc/asound.conf.original 

### 2.5 Create /etc/asound.conf

This step will make the virtual audio interfaces we create in PulseAudio available to applications that can't use PulseAudio directly, like Direwolf.  

As sudo, edit or create /etc/asound.conf in a text editor.  It should contain only the following lines.  

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

### 2.6 Modify /etc/pulse/default.pa

The changes to this file will set up the Fe-Pi left and right channels and tell PulseAudio to use the NULL audio source (no input audio) and the Pi's built-in sound card for the audio sink (output).  

We will use environment variables __PULSE_SOURCE__ and __PULSE_SINK__ when starting fldigi so that it uses the Fe-Pi audio interface instead of NULL/built-in-audio-card.  

Likewise, we'll specify the left and right virtual interfaces we create in __default.pa__ below, in __direwolf.conf__ to point direwolf to the correct radio.  

As sudo, make a backup of /etc/pulse/default.pa in /etc/pulse/default.pa.original:

	cd /etc/pulse 
	sudo mv default.pa default.pa.original 

As sudo, edit /etc/pulse/default.pa so it looks like the following.  

	#!/usr/bin/pulseaudio -nF
	#
	# This file is part of PulseAudio.
	#
	# PulseAudio is free software; you can redistribute it and/or modify it
	# under the terms of the GNU Lesser General Public License as published by
	# the Free Software Foundation; either version 2 of the License, or
	# (at your option) any later version.
	#
	# PulseAudio is distributed in the hope that it will be useful, but
	# WITHOUT ANY WARRANTY; without even the implied warranty of
	# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
	# General Public License for more details.
	#
	# You should have received a copy of the GNU Lesser General Public License
	# along with PulseAudio; if not, see <http://www.gnu.org/licenses/>.

	# This startup script is used only if PulseAudio is started per-user
	# (i.e. not in system mode)

	.fail

	### Load audio drivers statically
	### (it's probably better to not load these drivers manually, but instead
	### use module-udev-detect -- see below -- for doing this automatically)
	load-module module-alsa-card device_id="Audio" name="fepi" card_name="fepi" namereg_fail=false tsched=yes fixed_latency_range=no ignore_dB=no deferred_volume=yes use_ucm=no rate=96000 format=s32le source_name="fepi-capture" sink_name="fepi-playback" mmap=yes

	#  Create the left source/sink
	load-module module-remap-sink sink_name="fepi-playback-left" master="fepi-playback" channels=1 channel_map="mono" master_channel_map="front-left" remix=no
	load-module module-remap-source source_name="fepi-capture-left" master="fepi-capture" channels=1 channel_map="mono" master_channel_map="front-left" remix=no

	#  Create the right source/sink
	load-module module-remap-sink sink_name="fepi-playback-right" master="fepi-playback" channels=1 channel_map="mono" master_channel_map="front-right" remix=no
	load-module module-remap-source source_name="fepi-capture-right" master="fepi-capture" channels=1 channel_map="mono" master_channel_map="front-right" remix=no

	#  Load up the onboard audio and use it for a monitor channel
	.nofail
	load-module module-alsa-card device_id="ALSA" name="system-audio" card_name="system-audio" namereg_fail=false tsched=yes fixed_latency_range=no ignore_dB=no deferred_volume=yes use_ucm=no rate=48000 format=s32le sink_name="system-audio-playback" mmap=no

	# Use only ONE of the following load-module lines.  The module-loopback line will monitor audio from the radio on the Pi's speakers
	# The module-combine-sink will monitor, on the Pis speakers, audio sent by the Pi to the radio 
	#load-module module-loopback source="fepi-capture" sink="system-audio-playback"
	#load-module module-loopback source="fepi-playback-monitor" sink="system-audio-playback"
	load-module module-combine-sink sink_name=combined slaves=fepi-playback,system-audio-playback channels=2
	.fail
	#

	### Load several protocols
	.ifexists module-esound-protocol-unix.so
	load-module module-esound-protocol-unix
	.endif
	load-module module-native-protocol-unix

	### Automatically move streams to the default sink if the sink they are
	### connected to dies, similar for sources
	load-module module-rescue-streams

	### Make sure we always have a sink around, even if it is a null sink.
	load-module module-always-sink

	### Honour intended role device property
	load-module module-intended-roles

	### Automatically suspend sinks/sources that become idle for too long
	load-module module-suspend-on-idle

	### If autoexit on idle is enabled we want to make sure we only quit
	### when no local session needs us anymore.
	.ifexists module-console-kit.so
	load-module module-console-kit
	.endif
	.ifexists module-systemd-login.so
	load-module module-systemd-login
	.endif

	### Make some devices default
	#set-default-sink combined
	#set-default-sink fepi-playback
	#set-default-source fepi-capture
	set-default-sink system-audio-playback
	set-default-source null

### 2.7 Prevent pulseaudio from being the default sound interface  

Make a copy of /usr/share/alsa/pulse-alsa.conf:  

	sudo cp /usr/share/alsa/pulse-alsa.conf /usr/share/alsa/pulse-alsa.conf.original  

As sudo, edit /usr/share/alsa/pulse-alsa.conf and comment out all lines so it looks like this:

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

### 2.8 Restart pulseaudio.  Open a terminal, then:

	pulseaudio -k 
	pulseaudio --start --log-target=syslog

## 3. Scenarios

At this stage, PulseAudio is ready to use on either the left or the right radio or both at once.  

Decide how you want to use your radios and Fldigi and Direwolf.  

1. Determine where your radio(s) is/are connected:

	- The "Left" radio (uses GPIO 12 for PTT) is plugged in to the left 
6-pin miniDIN **_or_** the RJ45 jack
	- The "Right" radio (uses GPIO 23 for PTT) is plugged in to the right 
6-pin miniDIN **_or_** the TRRS jack.

2. Select your desired scenario from the next Scenario sections.

### 3.1 Scenario 1: I only want to use Fldigi with one radio.

#### 3.1.1 Configure Fldigi

1. Open Fldigi, click __Configure > Sound Card__.  Select the __Devices__ tab.
1. Check _PulseAudio_ and leave the *Server String* field empty. 
1. Select the __Settings__ tab and select Converter __Medium Sinc Interpolator__ (best for V/UHF - change as desired) 
1. Select the __Right channel__ tab.  
	For "Right" connected radios: Check _Reverse left/right channels_ under 
__Transmit__ Usage *and* under __Receive__ Usage.  
  For "Left" connected radios, leave these 2 boxes unchecked. 
1. Click the Rig tab then the GPIO tab and set your GPIO to whatever your radio uses: 
  	- "Left" radios: _BCM 12_, also check the corresponding _= 1 (on)_ box  
  	- "Right" radios: _BCM 23_, also check the corresponding _= 1 (on)_ box
1. Click __Save__ and __Close__
1. Restart Fldigi

### 3.2 Scenario 2: I only want to use Direwolf with one radio.

#### 3.2.1 Configure Direwolf

In your direwolf.conf file (usually located in your home folder
or in /etc), change the ADEVICE line to look like this:

	ADEVICE fepi-capture-xxx fepi-playback-xxx

where xxx = left or right depending on where your radio is plugged in to the DigiLink

Also change the PTT line to 

	PTT GPIO y

where y = 12 for the left radio or 23 for the right radio.

#### 3.2.2 Restart Direwolf using whatever script you use to operate direwolf.

### 3.3 Scenario 3: I want to run Fldigi on the {right or left} radio and Direwolf on the {left or right} radio at the same time.  
                
*Note that if Direwolf is on the Left radio, then Fldigi must be on the Right radio and vice-versa.  Fldigi and Direwolf cannot both use a single radio at the same time.  Likewise, two instances of Fldigi cannot use the same radio at the same time.*

#### 3.3.1 Follow the directions in Scenarios 1 and 2.

### 3.4 Scenario 4: I want to run 2 Fldigi applications at the same time, one for a "Right" radio and one for a "Left" radio.

#### 3.4.1 Make a copy of the fldigi configuration folder.  

Open a terminal, then run:

	cd ~
	cp -r .fldigi/ .fldigi2/

#### 3.4.2 Locate your fldigi.desktop file.

1. Click the Raspberry icon on the upper right of the desktop. 
1. Click the top menu containing your fldigi app (differs depending on your menu setup) 
1. _RIGHT_-click on Fldigi, then click Properties.  Note the "Target file". It will likely 
  be /home/pi/.local/share/applications/fldigi.desktop or 
  /usr/local/share/applications/fldigi.desktop or /home/pi/.local/share/applications/fldigi.desktop.  
  
  	Note the folder name containing fldigi.desktop.
1. Open a terminal.  Go to the folder containing fldigi.desktop (we'll use 
  __/home/pi/.local/share/applications__ in this example).

		cd ~/.local/share/applications/

#### 3.4.3 Make new menu entries for each radio.

1) Example: Say your 2 radios are a Kenwood (on the left) and an Alinco (on the
right), then:

		cp fldigi.desktop fldigi-kenwood.desktop 
		cp fldigi.desktop fldigi-alinco.desktop 
		mv fldigi.desktop fldigi-alinco.desktop-original

1) Edit the fldigi-kenwood.desktop file, changing the Exec and Name
lines as follows:

		Exec=sh -c 'PULSE_SINK=fepi-playback PULSE_SOURCE=fepi-capture fldigi --config-dir=/home/pi/.fldigi -title "Kenwood fldigi"' 
		Name[en_US]=Fldigi - Kenwood

1) Edit the fldigi-alinco.desktop file, changing the Exec and Name
lines as follows:

		Exec=sh -c 'PULSE_SINK=fepi-playback PULSE_SOURCE=fepi-capture fldigi --config-dir=/home/pi/.fldigi2 -title "Alinco fldigi"' 
		Name[en_US]=Fldigi - Alinco

1) Reload the menu items.  Run this command in a terminal:

		lxpanelctl restart

	You should now see 2 Fldigi entries in your menu.

1) Run ONE of the new fldigi menu items (only run one at a time during setup!) and set up the sound card and Rig control settings as described in Scenario 1.  Close that instance of Fldigi and repeat for the second Fldigi menu item.
       
1) Now you can run both the Alinco and Kenwood Fldigis at the same time. 

## 4. Adjust audio settings.  

### Start alsamixer
Open a terminal and run:

	alsamixer

Use the arrow keys to select the control.  Notes on using alsamixer:

1. The default card should be PulseAudio.  If not, press F6 and select PulseAudio. 
1. Press F5 to show both capture and playback. Set MASTER at 100%.  Set Capture to 7. Some controls are in stereo.  The up and down arrows change the level of both left channels (for the left radio) and right (for the right radio).
      
	Pressing __Q__ or __E__ increases the left or right channel (radio) level respectively. 
	Pressing __Z__ or __C__ decreases the left or right channel (radio) level respectively.

1. Press __F6__ and select __0 bcm2835 ALSA__ from the list.   Set PCM to _100_.
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

	The remaining settings don't matter.  
	
	You will want to open alsamixer while running Fldigi and/or direwolf and adjust
these settings on the "1 Fe-Pi Audio" device as needed: 

	Capture (for audio coming from the radio into the Pi - radio RX)
	PCM (for audio coming from the Pi to the radio - radio TX)

	Once you're happy with your audio settings, press Esc to exit alsamixer.  

1. Save Audio Settings (usually not required)
	
	These audio settings should save automatically, but for good measure store them again by running:

		sudo alsactl store

	If you want to save different audio level settings for different scenarios, 
you can run this command to save the settings (choose your own file name/location):

		sudo alsactl --file ~/mysoundsettings1.state store

	...and this command to restore those settings:

		sudo alsactl --file ~/mysoundsettings1.state restore

## 5. (Optional) Monitor Radio's TX and/or RX audio on Pi's Speakers

If you have speakers connected to the Pi, you can configure PulseAudio to monitor the audio to and/or from the radio

To adjust the level of the audio on your Pi's speakers, use the speaker's volume knob if it has one.  The speaker icon in the upper right of the Pi desktop also controls the Pi's speaker volume.  
Third way: Adjust the volume in alsamixer (__0 bcm2835 ALSA__ device) or clicking the Raspberry icon on the desktop, select Preferences > Audio Device Settings.  
Select the __bcm2835 ALSA__ sound card, and adjust the slider to your liking.  

__DO NOT__ click the __Make Default__ button in Audio Device Settings!  Likewise, __DO NOT__ right-click on the Speaker icon in the upper right of the desktop and then click any  of the cards listed.  Doing so will change the default card from PulseAudio and will mess up your audio. To restore, close all instances of fldigi and direwolf and restart pulseaudio in a terminal:

	pulseaudio -k 
	pulseaudio --start --log-target=syslog
	
### 5.1 Control TX and/or RX monitoring control from the command line

These commands will enable and disable monitoring of the radio's TX and RX on the Pi's speakers.  None, one or both of these monitors can be run at the same time.

- TX monitoring  

		pactl load-module module-loopback source="fepi-playback.monitor" sink="system-audio-playback"
- RX monitoring  

		pactl load-module module-loopback source="fepi-capture" sink="system-audio-playback"
- Stop all monitoring

		pactl unload-module module-loopback

### 5.2 (Optional) Make some menu items to control RX and TX monitoring

You can create menu items to control RX and TX monitoring by placing the above commands in a \*.desktop file.
You may need to modify __Categories=__ in the files described below depending on how you have your Raspberry Pi menus set up.

1.	Create a file called ~/.local/share/applications/radio-monitor-rx.desktop with this text:

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

1.	Create a file called ~/.local/share/applications/radio-monitor-tx.desktop with this text:

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
		
1.	Create a file called ~/.local/share/applications/radio-monitor-stop.desktop with this text:

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
		
Edit the placement of these menu items as desired using the Pi's Main Menu editor (__Pi > Preferences > Main Menu Editor__)



