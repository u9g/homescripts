# Caddy Proxy Server

## Caddy Plex Configuration

I use [Caddy](https://github.com/mholt/caddy) as a proxy server and if you want to use it with Plex, there is a very simple config below that I used. I have since removed this from my config to help simplify things.

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

### Plex Configuration

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