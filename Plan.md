# TinyNAS Build Guide
 
A guide for building a small, low-cost NAS running Jellyfin, Sonarr, Radarr, Prowlarr, and qBittorrent on a Libre Computer Renegade board (or similar single-board computer), with a hardened, non-root setup.
 
## Part 1: Physical Assembly
 
**Step 1: Attach the heatsink**
 
Attach a heatsink to the board's main chip and RAM chip following your specific heatsink kit's instructions.
 
**Step 2: Wire the fan**
 
1. Identify Pin 4 (5V) and Pin 6 (GND) on the board's GPIO header. These are standard Raspberry Pi 3 compatible locations.
2. Connect the fan's red wire to Pin 4 and the black wire to Pin 6.
**Step 3: Assemble the storage enclosure**
 
Insert your storage drive into its enclosure or case following the manufacturer's instructions. Do not plug in power yet.
 
## Part 2: Flash the OS
 
**Step 4: On your computer**
 
Download the Debian 12 or Ubuntu image built for your board and flash it onto a microSD card.
 
## Part 3: First Boot
 
**Step 5: Plug things in, in this order**
 
1. Insert the flashed microSD card into the board's microSD slot.
2. Plug an Ethernet cable from the board into your router.
3. Plug in your main storage drive (an NVMe SSD, SATA SSD, or similar external drive).
4. Plug in the power supply last. Most boards boot automatically once powered on. Some boards (including some Raspberry Pi models) require a physical power button press, so check your specific board's documentation if it doesn't boot on its own.
**Step 6: Find its IP address**
 
1. Log into your router's admin page. This is usually at 192.168.1.1 or 192.168.0.1.
2. Find the connected devices list and note your board's IP address (for example, 192.168.1.42).
**Step 7: SSH in as root, one time only**
```bash
ssh root@your-board-ip
```
Root access is only used for this initial setup. Step 13 replaces it with a proper user account.
 
## Part 4: Security Hardening
 
**Step 8: Update the system**
```bash
apt update && apt upgrade -y
```
 
**Step 9: Create a non-root user**
```bash
adduser tinynas
```
You can replace "tinynas" with any username you like. If you do, use that same name in place of "tinynas" throughout the rest of this guide.
 
Follow the prompts to set a strong password, then give the account admin rights:
```bash
usermod -aG sudo tinynas
```
 
**Step 10: Note this user's IDs**
 
You'll need these later.
```bash
id -u tinynas
id -g tinynas
```
These are almost always both 1000 for the first user created on a fresh system, but note the actual numbers your system returns.
 
**Step 11: Set up SSH key login (recommended, optional)**
 
There are two ways to log in going forward: a password, or a cryptographic key.
 
A password is something you memorize and type each time. Its security depends entirely on how strong and unique it is. A weak or reused password can be guessed or brute-forced.
 
A cryptographic key works differently. Instead of memorizing something, your computer holds a long, effectively unguessable private key file. When you connect, your computer proves it has the matching key through math, and nothing that could be intercepted and reused is ever transmitted. An attacker would need to physically steal the key file itself, not just guess a string of characters.
 
For a setup that stays on your local network only, with no internet exposure, a strong password is genuinely sufficient. The realistic threat model doesn't include someone brute-forcing SSH from outside your house, since nobody outside can reach it.
 
That said, keys are strictly more secure with almost no downside once set up, and typing a password every time goes away entirely. If you don't mind one extra setup step, it's worth doing. If you'd rather keep things simple, a strong password is a completely reasonable choice.
 
If you want to set up a key, run these two commands on your own computer, not on the board:
```bash
ssh-keygen -t ed25519
ssh-copy-id tinynas@your-board-ip
```
Follow the prompts, entering the tinynas password when asked. This lets you log in without typing a password each time.
 
**Step 12: Disable root SSH login**
 
Back on the board, still logged in as root:
```bash
nano /etc/ssh/sshd_config
```
Find the line starting with PermitRootLogin and change it to:
```
PermitRootLogin no
```
Save and exit, then restart SSH:
```bash
systemctl restart sshd
```
 
**Step 13: Log out and back in as the new user**
```bash
exit
```
```bash
ssh tinynas@your-board-ip
```
From here on, every command in this guide runs from this session, using sudo where admin privileges are needed. See the "Using sudo" section at the end of this guide if you're unfamiliar with it.
 
## Part 5: Storage Setup
 
**Step 14: Check if a partition already exists**
```bash
lsblk
```
If you see your drive (commonly listed as /dev/sda) with no /dev/sda1 underneath it, continue to Step 15. If /dev/sda1 already exists, skip ahead to Step 16.
 
**Step 15: Create a partition, only if needed**
```bash
sudo parted /dev/sda --script mklabel gpt mkpart primary ext4 0% 100%
```
Confirm it worked:
```bash
lsblk
```
 
**Step 16: Format and mount the drive**
```bash
sudo mkfs.ext4 /dev/sda1
sudo mkdir -p /mnt/storage
sudo mount /dev/sda1 /mnt/storage
sudo mkdir -p /mnt/storage/media
sudo mkdir -p /mnt/storage/media/downloads
```
Find the drive's UUID:
```bash
sudo blkid /dev/sda1
```
Open the filesystem table:
```bash
sudo nano /etc/fstab
```
Add this line at the bottom, replacing YOUR-UUID-HERE with the UUID you just found:
```
UUID=YOUR-UUID-HERE  /mnt/storage  ext4  defaults  0  2
```
Save and exit, then test it with these three separate commands:
```bash
sudo umount /mnt/storage
```
```bash
sudo mount -a
```
```bash
df -h /mnt/storage
```
You should see the drive listed with available space.
 
**Step 17: Set ownership to your new user**
 
Replace 1000:1000 with the actual numbers from Step 10 if they differ.
```bash
sudo chown -R 1000:1000 /mnt/storage/media
```
 
## Part 6: Install Docker
 
**Step 18: Install Docker**
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker tinynas
```
Log out and back in for the group change to apply:
```bash
exit
```
```bash
ssh tinynas@your-board-ip
```
 
**Step 19: Redirect Docker's storage off the microSD card**
```bash
sudo systemctl stop docker
sudo mkdir -p /mnt/storage/docker
sudo nano /etc/docker/daemon.json
```
Add the following:
```json
{
  "data-root": "/mnt/storage/docker"
}
```
Save and exit, then restart Docker:
```bash
sudo systemctl start docker
docker info | grep "Docker Root Dir"
```
Confirm the output shows /mnt/storage/docker. This keeps Docker's heavy, constant write activity on the SSD instead of wearing out the microSD card.
 
## Part 7: Deploy Jellyfin, Sonarr, Radarr, qBittorrent, and Prowlarr
 
These five services work together as a pipeline. Sonarr watches for new episodes of TV shows or anime you tell it to track, and automatically searches for and grabs them once available. Radarr does the same job for movies: you tell it a movie you want, and it finds and grabs it. Prowlarr is the shared search directory both of them use; you add your torrent indexer sites into Prowlarr once, and it automatically feeds that list into both Sonarr and Radarr, so indexers only need to be configured in one place. qBittorrent is the actual download engine: once Sonarr or Radarr finds a match through Prowlarr, qBittorrent downloads the file. Put together, Prowlarr finds where something exists, Sonarr or Radarr decides what to grab and when, qBittorrent downloads it, and the finished file lands in the media folder where Jellyfin picks it up automatically.
 
**Step 20: Create the project folders**
```bash
mkdir -p ~/nas/config/jellyfin ~/nas/config/sonarr ~/nas/config/radarr ~/nas/config/qbittorrent ~/nas/config/prowlarr
cd ~/nas
```
 
**Step 21: Create the compose file**
```bash
nano docker-compose.yml
```
```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
    ports:
      - "8096:8096"
    volumes:
      - ~/nas/config/jellyfin:/config
      - /mnt/storage/media:/media
    restart: unless-stopped
 
  sonarr:
    image: linuxserver/sonarr
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
    ports:
      - "8989:8989"
    volumes:
      - ~/nas/config/sonarr:/config
      - /mnt/storage/media:/media
    restart: unless-stopped
 
  radarr:
    image: linuxserver/radarr
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
    ports:
      - "7878:7878"
    volumes:
      - ~/nas/config/radarr:/config
      - /mnt/storage/media:/media
    restart: unless-stopped
 
  qbittorrent:
    image: linuxserver/qbittorrent
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - WEBUI_PORT=8080
    ports:
      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"
    volumes:
      - ~/nas/config/qbittorrent:/config
      - /mnt/storage/media/downloads:/downloads
    restart: unless-stopped
 
  prowlarr:
    image: linuxserver/prowlarr
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
    ports:
      - "9696:9696"
    volumes:
      - ~/nas/config/prowlarr:/config
    restart: unless-stopped
```
Save and exit. Replace 1000 with your actual numbers from Step 10 if they differed.
 
**Step 22: Start everything**
```bash
docker compose up -d
docker ps
```
All five containers should show as "Up".

 
**Step 23: Finish setup in your browser**
 
Visit each of these from a browser on your computer, replacing the IP with your board's own:
- http://your-board-ip:8096 for the Jellyfin setup wizard (set the library path to /media)
- http://your-board-ip:8989 for Sonarr setup
- http://your-board-ip:7878 for Radarr setup
- http://your-board-ip:8080 for qBittorrent
- http://your-board-ip:9696 for Prowlarr setup
  
**Step 24: Change qBittorrent's default password immediately**
 
The default login is admin and adminadmin. Log in once, then go to Tools, then Options, then Web UI, and set a new, strong password before doing anything else.
 
**Step 25: Set up Prowlarr**
 
1. Open Prowlarr in your browser.
2. Add indexers: go to Indexers, then Add Indexer, and pick one or two from Prowlarr's built-in list of known public torrent indexers, then save.
3. Get Sonarr's API key: open Sonarr, go to Settings, then General, and copy the API key shown there.
4. Get Radarr's API key the same way, from Radarr's Settings, then General page.
5. Back in Prowlarr, go to Settings, then Apps, then Add Application, choose Sonarr, enter the address http://sonarr:8989, and paste in Sonarr's API key, then save.
6. Repeat for Radarr, using address http://radarr:7878 and Radarr's API key.
Once both are added, Prowlarr automatically pushes its indexer list into both Sonarr and Radarr. They'll appear in each app's own indexer settings without needing to be added manually there.
 
**Step 26: Set up qBittorrent**
 
If you haven't already changed the password in Step 24, do so now under Tools, then Options, then Web UI.
 
**Step 27: Set up Sonarr**
 
1. Open Sonarr in your browser.
2. Set a root folder: go to Settings, then Media Management, then Root Folders, then Add, and enter /media/tv.
3. Optionally add a second root folder for anime, using the same menu path and entering /media/anime. This lets you choose which folder a show goes into when you add it, rather than everything landing in one shared folder.
4. Set up the download client: go to Settings, then Download Clients, then Add, then qBittorrent. Enter the address qbittorrent, port 8080, and the username and password from Step 24, then test and save.
5. Set the quality profile: go to Settings, then Profiles, edit your profile, and uncheck anything above 1080p, then save.
**Step 28: Set up Radarr**
 
1. Open Radarr in your browser.
2. Set a root folder: go to Settings, then Media Management, then Root Folders, then Add, and enter /media/movies.
3. Set up the download client the same way as Step 27, using the same qBittorrent details.
4. Set the quality profile the same way, capping it at 1080p.

**Step 29: Create matching libraries in Jellyfin**
 
1. Open Jellyfin, go to Dashboard, then Libraries, then Add Media Library.
2. Add one library pointed at /media/tv with content type "Shows."
3. Add another pointed at /media/movies with content type "Movies."
4. If using a separate anime folder, add a third library pointed at /media/anime. Jellyfin doesn't have a distinct anime content type built in, so use "Shows" here as well, and select an anime-style metadata option if your Jellyfin version offers one.
 
**Step 30: How to add something going forward**
 
To add a movie, open Radarr, click Add New Movie, search the title, select the correct match, confirm the root folder and quality profile, and click Add Movie. It searches and downloads automatically.
 
To add a TV show, open Sonarr, click Add New Series, search the title, select the correct match, choose the appropriate root folder, confirm the quality profile, and choose whether to monitor all episodes or only future ones, then click Add Series.
 
That covers the whole loop after initial setup. Prowlarr, qBittorrent, and Jellyfin need no further action per item. Adding something in Sonarr or Radarr is the only step that repeats.
 
## Part 8: Confirm Nothing Is Exposed to the Internet
 
**Step 31: Check your router**
 
Log into your router's admin page and check for the following.
 
Port forwarding rules: confirm none of ports 8096, 8989, 7878, 8080, or 9696 are forwarded to the NAS from the internet.
 
UPnP: if enabled, some applications, particularly torrent clients, can silently ask the router to open ports on their own. Consider disabling UPnP on your router entirely, or at minimum review its list of currently open ports after setup and remove anything unexpected.
 
This keeps the whole setup strictly local, reachable only by devices already on your home network.
 
## Part 9: Connect Kodi
 
**Step 32: Install the Kodi Sync Queue plugin in Jellyfin**
 
1. Open Jellyfin, go to Dashboard, then Plugins, then Catalog.
2. Find Kodi Sync Queue in the list and install it.
3. Restart the Jellyfin container to activate it:
```bash
docker restart jellyfin
```
This lets Kodi sync only what's changed since the last check instead of rescanning the whole library each time, which is worth having on lower-power boards.
 
**Step 33: Install the Jellyfin add-on in Kodi**
 
1. Boot into Kodi on whatever device you're using as a client (for example, a Batocera setup).
2. Go to the add-ons section, usually reached through a puzzle piece icon on the home menu, or through Settings, then Add-ons.
3. Select Install from repository, then Kodi Add-on repository, then Video add-ons.
4. Find and select Jellyfin for Kodi, then click Install, and wait for the confirmation notification.

**Step 34: Connect it to your NAS**
 
1. Once installed, find Jellyfin under Add-ons on Kodi's home screen (it may also add its own entry to the main menu).
2. Open it. You'll be prompted to add a server.
3. Choose manual entry rather than auto-discover, since auto-discover isn't always reliable on wired-only setups, and enter your NAS's address exactly as http://your-board-ip:8096.
4. Log in with the same username and password created during Jellyfin's setup wizard.

**Step 35: Set it as your default view, optional**
 
In Kodi's settings, you can set the Jellyfin add-on to launch automatically or pin it to the home screen, so you don't have to dig through menus each time. This is usually found under Settings, then Interface, then Skin, then Home Menu, though the exact path varies by Kodi skin and version.
 
**Step 36: Test playback**
 
1. Browse to any show or movie through the Jellyfin add-on and press play.
2. On your computer, check Jellyfin's web dashboard under the Activity tab during playback.
3. It should say "Direct Play." If it says "Transcode" instead, see Part 10 below.

## Part 10: Pre-Transcoding
 
Use this for files that won't direct play.
 
**Step 37: When to use this**
 
This step applies specifically to boards like the Renegade, which lack the hardware needed for smooth live transcoding. If you're using a computer with Intel Quick Sync or another hardware-accelerated setup that Jellyfin officially supports, you can skip this step entirely, since your hardware can transcode in real time without issue.

If you went the budget board route like this guide does, follow this step. Boards like the Renegade can transcode video, just not fast enough to keep up with live playback. Pre-transcoding gets around this: instead of converting a file the moment you press play (which is what causes stuttering on weaker hardware), you convert it ahead of time, whenever it's convenient, so playback afterward is instant with no live conversion needed at all.
 
**Step 38: Install ffmpeg**
```bash
sudo apt install ffmpeg -y
```
 
**Step 39: Convert a file to a widely compatible 1080p format**
```bash
ffmpeg -i /mnt/storage/media/input.mkv -vf scale=-2:1080 -c:v libx264 -crf 23 -c:a aac /mnt/storage/media/output.mkv
```
This can take a while on lower-power hardware. It's fine to let it run in the background since it doesn't need to keep pace with live playback.
 
**Step 40: Replace the original**
 
Once the converted file plays back correctly in a quick test, delete the original and keep the converted version.
 
## Part 11: Ongoing Care
 
**Keep your containers updated**
 
The operating system updates through apt, but Docker containers such as Jellyfin, Sonarr, Radarr, Prowlarr, and qBittorrent don't update automatically as part of that. They need their own separate update command. This matters for real security reasons: Jellyfin in particular has had genuine vulnerabilities patched in past versions, so staying current is worth doing periodically even on a setup with no internet exposure.
```bash
cd ~/nas
docker compose pull
docker compose up -d
```
This pulls the latest version of each image and restarts the containers with the update applied, without touching configuration or media. Running this roughly once a month is reasonable. There's no strict schedule required, just don't leave it untouched for a year or more.
 
**Always shut down properly**
```bash
sudo shutdown -h now
```
Wait for the lights to stop before unplugging power. This matters especially if your storage drive lacks power loss protection.
 
## Quick Reference: Ports
 
- Jellyfin: 8096
- Sonarr: 8989
- Radarr: 7878
- qBittorrent: 8080
- Prowlarr: 9696
## Quick Reference: Using sudo
 
Since root SSH login is disabled after Part 4, you'll be logged in as your regular user for every session. Any command that needs admin level permissions must be prefixed with sudo.
```bash
sudo <the command you want to run>
```
For example, this fails for a regular user:
```bash
apt update
```
This works:
```bash
sudo apt update
```
It will prompt for your user's password, not an SSH key passphrase if you're using one. Type it and press enter. A few things worth knowing:
 
It remembers you for about 15 minutes after the first password entry, so a string of admin commands run one after another will only ask once.
 
Not everything needs it. Commands like ls, cd, docker ps, and cat somefile.txt work fine without it. It's only needed for installing packages, editing system configuration files, mounting drives, or restarting system services, which is why it appears in front of specific commands throughout this guide.
 
If you forget it, no harm done. You'll just see a "Permission denied" message. Re-run the same command with sudo in front of it.
 
