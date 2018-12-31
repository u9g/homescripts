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

I use a combination of mergerfs and rclone to keep a local mount that is always written to first and my second mount point is my rclone Google Drive. These 2 mount points are unified at /gmedia (which is where the media is accessible to Plex, Emby, etc).

        /data/local (local mirror disk)
        /GD (rclone mount)
    /gmedia

[rclone.conf](https://github.com/animosity22/homescripts/blob/master/rclone.conf)

They all get mounted up via my systemd scripts for [gmedia-service](https://github.com/animosity22/homescripts/blob/master/rclone-systemd/gmedia.service).

My gmedia starts up items in order:
1) [rclone mount](https://github.com/animosity22/homescripts/blob/master/rclone-systemd/gmedia-rclone.service)
2) [mergerfs mount](https://github.com/animosity22/homescripts/blob/master/rclone-systemd/gmedia.mount)
3) [find command](https://github.com/animosity22/homescripts/blob/master/rclone-systemd/gmedia-find.service) which justs caches the directory and file structure and provides me an output of the structure. This is not required but something I choose to do to warm up the cache.

## [mergerfs@github](https://github.com/trapexit/mergerfs)

I use mergerfs over unionfs as it provides me the ability to define a file system to always write first to.

I use the following options:

```bash
Options = defaults,sync_read,allow_other,category.action=all,category.create=ff
```

Important items:

- sync_read as rclone is default built with this and is required for proper streaming
- category.action=all,category.create=ff says to always create directories / files on the first listed mount point and for my configuration that is my /data/mounts/local
- if you are reading directly from your rclone mount, you do not need to worry about any of the mergerfs settings.

## Scheduled Nightly Uploads

I moved my files to my GD every ngiht via a cron job and an [upload cloud](https://github.com/animosity22/homescripts/blob/master/scripts/upload_cloud) script. This leverages an [excludes](https://github.com/animosity22/homescripts/blob/master/scripts/excludes) file which gets rid of partials and my torrent directory.

## Plex and Caddy Configuration

I use [Caddy](https://github.com/mholt/caddy) as a proxy server and route all my items through it. I build via this [script](https://github.com/animosity22/homescripts/blob/master/scripts/build_caddy).

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

UFW or other firewall:
- Deny port 32400 externally (Plex still pings over 32400, some clients may use 32400 by mistake despite 443 and 80 being set).

<i>Note adding allowLocalhostOnly="1" to your Preferences.xml, will make Plex only listen on the localhost, achieving the same thing as using a firewall and this is what I use in my configuration.</i>

## Known Issues

- Plex Playback
  - Apple TV (4th generation)
    - Direct Play Stuttering
      - This <i>seems</i> to be fixed in IOS 12 but I still use a proxy in front of my plex server as I use [Caddy](https://github.com/mholt/caddy) as a proxy server.
- Plex Music
	- Music play for plex needs to use the cache backend as it repeatedly opens and closes files. The VFS backend will not work due to this limitation.
- Bazaar
	- This seems to create a lot of API hits. It currently does not support forced subs so I don't use it.
