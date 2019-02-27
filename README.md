# README.md

This is my configuration that works for me the best for my use case of an all in one media server. This is not meant to be a tutorial by any means as it does require some knowledge to get setup. I'm happy to help out as best as I can and welcome any updates/fixes/pulls to make it better and more helpful to others.

## Home Configuration

- Verizon Gigabit FIOS
- Google Drive with an Encrypted Media Folder
- Debian Stretch 9.5
- Intel(R) Core(TM) i7-7700 CPU @ 3.60GHz
- 32 GB of Memory
- 250 GB SSD Storage for my root
- 6TB mirrored for staging
- rTorrent, NZBGet, Sonarr, Radarr and Ombi all run locally on my mergerfs mount that allows hard linking of files.

## OpenVPN Configuration

For all my private traffic, I use [TorGuard](https://torguard.net/) as they support port forwarding and have very good support.

[Setup and Configuration](https://github.com/animosity22/homescripts/blob/master/OPENVPN.MD)

## Rclone configuration

### Installation

The first step is to get fuse installed and configured properly.

	sudo apt install fuse -y
	
You need to make the change to /etc/fuse.conf to allow_other by uncommenting the last line by and remove the # from the last line.

	sudo vi /etc/fuse.conf
	root@gemini:~# cat /etc/fuse.conf
	# /etc/fuse.conf - Configuration file for Filesystem in Userspace (FUSE)
	
	# Set the maximum number of FUSE mounts allowed to non-root users.
	# The default is 1000.
	#mount_max = 1000

	# Allow non-root users to specify the allow_other or allow_root mount options.
	user_allow_other
	
After fuse is installed, I install mergerfs as it's already part of the repositories for Debian.

	sudo apt install mergerfs -y

My use case for mergerfs is that I always want to write to the local disk first and all my applications (Sonarr/Radarr/rTorrent/Plex/Emby/etc) all point directly to /gmedia. For them it's not relevant if the file is local or remote as they should act the same.

  	/gmedia
        /data/local (local mirror disk)
        /GD (rclone mount)
  

My rclone.conf has an entry for the Google Drive connection and and encrypted folder in my Google Drive called "media". I mount media with rclone script to display the decrypted contents to my server. I do not use the cache backend as I just use the default VFS backend. I've found that Plex Music works better with the cache backend as that tends to open and close files a lot. If you are having problems, you can always try to the cache backend to see if that works better for your use case.

My rclone looks like: [rclone.conf](https://github.com/animosity22/homescripts/blob/master/rclone.conf)

They all get mounted up via my systemd scripts for [gmedia-service](https://github.com/animosity22/homescripts/blob/master/rclone-systemd/gmedia.service).

My gmedia starts up items in order:
1) [rclone mount](https://github.com/animosity22/homescripts/blob/master/rclone-systemd/gmedia-rclone.service)
2) [mergerfs mount](https://github.com/animosity22/homescripts/blob/master/rclone-systemd/gmedia.mount) This needs to be named the same as the mount point for the .mount to work properly. I use /gmedia so the file is named accordingly.
3) [find command](https://github.com/animosity22/homescripts/blob/master/rclone-systemd/gmedia-find.service) which justs caches the directory and file structure and provides me an output of the structure. This is not required but something I choose to do to warm up the cache.

### mergerfs configuration
This is located over here if you want to request help or compile from source [mergerfs@github](https://github.com/trapexit/mergerfs)

I found unionfs to not do what I wanted and I can't stand the hidden files so for my, it's much easier to configure and use mergerfs.

The following options make it always write to the first disk in the mount:

```bash
Options = defaults,sync_read,allow_other,category.action=all,category.create=ff
```

Important items:

- sync_read as rclone is default built with this and is required for proper streaming
- category.action=all,category.create=ff says to always create directories / files on the first listed mount point and for my configuration that is my /data/local
- if you are reading directly from your rclone mount, you do not need to worry about any of the mergerfs settings.

## Scheduled Nightly Uploads

I moved my files to my GD every ngiht via a cron job and an [upload cloud](https://github.com/animosity22/homescripts/blob/master/scripts/upload_cloud) script. This leverages an [excludes](https://github.com/animosity22/homescripts/blob/master/scripts/excludes) file which gets rid of partials and my torrent directory.

## Plex and Caddy Configuration

I use [Caddy](https://github.com/mholt/caddy) as a proxy server and route all my items through it. I build via this [script](https://github.com/animosity22/homescripts/blob/master/scripts/build_caddy). The purpose of the proxy server is to mask any client side issues of opening and closing of files as the Plex clients are very problematic with this. Caddy keeps the connection open and it makes playback seamless.

My plex configuration in my CaddyFile as follows:

```bash
# Plex Server
plex.animosity.us {
gzip
timeouts 1h
log /opt/caddy/logs/plex.log
tls {
  dns cloudflare
}
proxy / 127.0.0.1:32400 {
 transparent
 websocket
 keepalive 12
 timeout 1h
    }
}
```

Plex also needs to have a few additional steps to be added so Caddy is used in all situations.

### Plex

Remote Access - Disable

```
Network - Custom server access URLs = https://<your-domain>:443,http://<your-domain>:80
```
Network - Secure connections = Preferred

<i>Note: You can force SSL by setting required and not adding the HTTP URL, however some players which do not support HTTPS (e.g: Roku, Playstations, some SmartTVs) will no longer function. I only use SSL is my config and as I do not open up HTTP traffic. </i>

### Stopping Local Access
By default, Plex regardless of what override URL you set will still connect locally to 32400. To remove this, I use the second option of adding the option to the Preferences.xml. You need to stop Plex and add the line in near the end:

```bash
root@gemini:/var/lib/plexmediaserver/Library/Application Support/Plex Media Server# cat Preferences.xml
<?xml version="1.0" encoding="utf-8"?>
<Preferences OldestPreviousVersion="1.13.4.5271-200287a06" MachineIdentifier="4db8417f-c3c2-4aa8-96c4-267d0f4fa178" ProcessedMachineIdentifier="a2e448111539e64c746631ba21086e1951ed562f" AnonymousMachineIdentifier="5c03f573-ee73-4eaf-b15f-58743c756e78" GracenoteUser="WEcxAw1bZBMXkcb1506EOLokpBFOs7KyKYk3LnjfXyrpkYP4cBa8XPfV4w3Fy4syy0tv8UGntq6kn5kXBFayLnqwpafDPn2CK/fyMeAb/+EB3CAL2vD4CJxns1VwCa7ZM5jUXUiNZMHr9akNP/hAoGePyqJiAS5qLy8D5+dd+K10XJTB8DvXo2VtXIsN5gxeAKGw" MetricsEpoch="1" AcceptedEULA="1" FriendlyName="gemini" PublishServerOnPlexOnlineKey="0" PlexOnlineToken="d59epUrszcRB5HbUy7qs" PlexOnlineUsername="animosity022" PlexOnlineMail="earl.texter@gmail.com" LastAutomaticMappedPort="28408" CertificateVersion="2" PubSubServer="172.104.24.90" PubSubServerRegion="ewr" PubSubServerPing="51" ManualPortMappingMode="1" logDebug="0" ButlerUpdateChannel="8" FSEventLibraryPartialScanEnabled="1" LanNetworksBandwidth="192.168.1.0/24,127.0.0.1" allowedNetworks="192.168.1.99" secureConnections="1" TranscoderThrottleBuffer="600" HardwareAcceleratedCodecs="1" DlnaEnabled="0" ButlerDatabaseBackupPath="/data/backups/plexdb" ButlerTaskDeepMediaAnalysis="0" ButlerTaskGenerateAutoTags="0" ButlerTaskRefreshEpgGuides="0" ButlerTaskRefreshLibraries="1" ButlerTaskRefreshPeriodicMetadata="0" ButlerTaskReverseGeocode="0" LanguageInCloud="1" GenerateChapterThumbBehavior="never" LoudnessAnalysisBehavior="never" ScannerLowPriority="1" ScheduledLibraryUpdateInterval="21600" ScheduledLibraryUpdatesEnabled="1" autoEmptyTrash="0" ButlerEndHour="6" ButlerStartHour="2" customConnections="https://plex.animosity.us:443" CloudSyncNeedsUpdate="0" DlnaReportTimeline="0" CinemaTrailersFromLibrary="0" ButlerTaskUpgradeMediaAnalysis="0" FSEventLibraryUpdatesEnabled="0" ServerBindInterface="enp2s0" TreatWanIpAsLocal="0" GdmEnabled="0" BackgroundQueueIdlePaused="0" OnDeckWindow="8" LogVerbose="0" logTokens="0" allowLocalhostOnly="1" ManualPortMappingPort="443" GenerateBIFBehavior="asap" TranscoderTempDirectory=""/>
```

UFW or other firewall:
- Deny port 32400 externally (Plex still pings over 32400, some clients may use 32400 by mistake despite 443 and 80 being set).

<i>Note adding allowLocalhostOnly="1" to your Preferences.xml, will make Plex only listen on the localhost, achieving the same thing as using a firewall and this is what I use in my configuration.</i>
