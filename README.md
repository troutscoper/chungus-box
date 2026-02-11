# ChungusBox

ChungusBox is the name for my home Jellyfin server that is automated through the *arr stack with downloads handled by RDTClient. I use Homarr as a home page and also run a few other applications to make it look pretty, have more information, and mess around with. This isn't a perfect server by any means, but it fits exactly what I wanted, and works well if you don't intend to hoard media as it has limited drive space. I set this up over a weekend at the beginning of this year as a fun project, and it also saves money.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/0f319a75-c376-476a-8b8b-60554b05b381" />

# Hardware

ChungusBox is running on a refurbished Dell Optiplex 3050 SFF. These can be found for anywhere from $50-$200 dollars, at varying hardware levels.

The main thing is getting a 7th Gen or newer Intel Processor as a requirement for Jellyfin hardware transcoding. Even this isn't a hard requirement for general functionality, but it is recommended!

My specs are:

- Intel Core i5-7500 @ 3.8 GHz
- 16GB DDR4 @ 2133 MHz
- No-name 256GB SSD for boot
- Seagate 8TB HDD

In the future the goal is to have this machine serve as a brain hooked up to a NAS, for now it serves it's purpose.

# App Stack

All of my applications are containerized using [Docker](https://www.docker.com/). My Docker visualization app of choice is [Portainer](https://www.portainer.io/).

As far as home dashboards, there are tons of options depending on the amount of customization you want, and how comfortable you are with config files. Personally, I like GUI editing of the dashboard, so I use [Homarr](https://homarr.dev/). The main disadvantage of Homarr is how memory intensive it is. My dashboard eats up around 700MB of RAM, but the rest of my stack doesn't need a lot of memory to function. Keep that in mind if you plan on running any applications that require more RAM, or if you have a smaller amount available on your system.

For the media side of things specifically I use the following apps.

- [Jellyfin](https://jellyfin.org/) - Final Media Viewer
- [Jellyseerr](https://github.com/seerr-team/seerr) - Request/Discovery
- [Radarr](https://radarr.video) - Movie Downloads
- [Sonarr](https://sonarr.tv/) - TV/Anime Downloads
- [Prowlarr](https://prowlarr.com/) - Download Indexer
- [RDTClient](https://github.com/rogerfar/rdt-client) - Downloader
- [Byparr](https://github.com/ThePhaseless/Byparr) - Cloudflare Bypass

Here is a diagram of how all these apps interact with one another during a regular user flow.

```
[ User ]
   │
   │ (1) Request Content
   ▼
[ Jellyseerr ]
   │
   │ (2) Send "Monitor" Command
   ▼
[ Sonarr / Radarr ] ──────────────────────────────────────────┐
   │    │                                                     │
   │    │ (3) Search                                          │ (9) Import
   │    ▼                                                     │     & Rename
   │  [ Prowlarr ] <──> [ Byparr ] <──> [ Public Indexers ]   │
   │       │                                                  │
   │       │ (4) Return Results                               │
   │       ▼                                                  │
   │  (5) Send Magnet Link                                    │
   ▼                                                          │
[ RDTClient ]                                                 │
   │                                                          │
   │ (6) Offload to Cloud                                     │
   ▼                                                          │
[ Debrid Service ]                                            │
   │                                                          │
   │ (7) Download via HTTPS                                   │
   ▼                                                          │
[ /downloads Folder ] ────────────────────────────────────────┘
   │
   │ (8) File Moved
   ▼
[ /media Folder ]
   │
   │ (10) Read File
   ▼
[ Jellyfin ]
   │
   │ (11) Stream
   ▼
[ User ]
```

All the Docker Compose files for necessary applications for the Media Stack to work, the only application listed above that doesn't have a corresponding Docker Compose below is Homarr, as dashboards are very much up to personal preferance.

# Operating System

I used Ubuntu Desktop 24.04.3, if I was going to do it again I would probably just run Ubuntu server, but desktop is helpful for first time setup if you aren't comfortable in Linux terminal and want to check your work via GUI. After original setup, I've primarily interacted with it via SSH and the applications it runs.

<img width="705" height="359" alt="image" src="https://github.com/user-attachments/assets/8f4b8dfa-66b4-49aa-8939-55c133d71aae" />

You could theoretically run this on Windows but the entire workflow would be almost certainly different and you'd be on your own.

# Setting up the Essentials

I am not someone who would claim to have any Linux knowledge. After putting this together, I can confidently download things that work on the first try and have good documentation, and solidly navigate the Linux file structure via terminal. Hopefully, those are also the experiences you gain by following along. I'm also vaguely remembering the steps I took at this part. I trust that you, dear reader, can run basic Linux cleanup and lookup how to install something if it says it's not installed. There is plenty of guides online that are much more qualified than me, I'm sure I missed steps in my set-up during this part as well.

However, make sure you at least do this. I know you need these packages.

```
sudo apt install -y curl wget git unzip zip nano vim
```

## Static IP

First thing has nothing to do with Linux. Give your server a DHCP reservation to keep a static IP. This is different depending on your ISP/router, but all of them can do it. I gave mine the IP of 192.168.1.67 because I'm horribly brain-rotted and this machine is called ChungusBox.

If you don't know how to find the MAC address:


```
ip link
```


Technically, you should keep your static IP ending as whatever it's set as but I don't care haha 6-7 funny.

## Docker

Everything else from this point is going to essentially run through Docker. It's super easy to set up from here (for the most part).

1. Set up Docker's apt repository


```
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

```

2. Install Docker Packages


```
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

3. Check that it worked.


```
sudo docker run hello-world
```


### Portainer

This will be your first browser app and is used as a visualization tool for Docker. I don't use Docker in the terminal as I'm too lazy to figure out how, but it isn't difficult and you could do it if you'd like to. Here's how to install Portainer though.

1. Create Portainer volume

```
docker volume create portainer_data
```

2. Run Portainer container

```
docker run -d \
  --name portainer \
  --restart=always \
  -p 8000:8000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

3. Access Portainer from any device on LAN @ https://SERVER-IP:9443

5. Do your first time set-up!

# File System

All of my config files live in the /srv/ directory within a folder for their specific app, example below.


```
/srv/jellyfin/config
```

The rest of the everything lives on an 8TB Seagate that is mounted to /chungus/, aptly named. Here is the basic file structure for that. The easiest way to follow along would be to map your drive to something and work off that folder directly. It doesn't have to be /chungus/, but if you change it to /boomerang/, you only have to change the instances of chungus, not everything entirely.

As far as /srv/, that's the standard place for config files serving the web, hence the s(e)rv(e) nature of it.

```
├── downloads                                                                                                                                            
│   └── rdtclient                                                                                                                                          
│       ├── radarr                                                                                                                                         
│       ├── testDefault.rar                                                                                                                                 
│       └── tv-sonarr                                                                                                                                       
└── media                                                                                                                                                  
    ├── movies                                                                                                                                                     
    ├── music                                                                                                                                               
    └── tv                                                                                                            
```

The pipeline for what we're about to create is basically this.

```
Downloaded to /chungus/downloads --> Moved by Radarr/Sonarr to correct /chungus/media/... --> Picked up by Jellyfin
```

# Minimum Functionality

Pretty much everything from this point on is just the Docker Compose files. You can access the section to add these in Portainer by going to your Local Environments > Stacks > Add Stack. Here you can paste your Docker Compose files.

Below is going to be direct rips of my Docker Composes. Pretty much everything can be kept the same, but specifically your volume mappings may have to be different. If you wanna copy mine, that's totally fine. You could also put all of these into a singular stack, I have them seperate because that's what I have. 

These are apps required for the one click download workflow. Simply copy and paste these Docker Composes where they belong and deploy the stack. Each app can be accessed via the static-ip you set with the ports set within the file under ports.

Pay attention to the volume mapping, I'd recommend reading into it more, but in short the left side is your computer, the right side is what docker sees. It's important to have many of these mapped to similar locations, like radarr and rdt-client.

## Jellyfin

Jellyfin is an open-source version of Plex essentially. It's a media viewer that isn't bogged down by ads or any sponsored content. You can use Plex if you'd like to, the set-up would be a bit different however, and I wouldn't recommend it with the direction Plex is going.

```
version: "3.8"

services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    ports:
      - "8096:8096"
    volumes:
      - /srv/jellyfin/config:/config
      - /chungus/media:/media
    environment:
      - TZ=America/New_York

```

## Jellyseerr

Jellyseerr is a media discovery and requesting tool. It can be used by end-users to easily start a new movie down the download pipeline.

```
---
services:
  jellyseerr:
    image: ghcr.io/fallenbagel/jellyseerr:latest
    init: true
    container_name: jellyseerr
    environment:
      - LOG_LEVEL=debug
      - TZ=Asia/Tashkent
      - PORT=5055 #optional
    ports:
      - 5055:5055
    volumes:
      - /srv/jellyseerr/config:/app/config
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:5055/api/v1/status || exit 1
      start_period: 20s
      timeout: 3s
      interval: 15s
      retries: 3
    restart: unless-stopped
```

## Radarr

Radarr is used to download our movies specifically. It is the tool that will deal with actually picking out the file to be downloaded and moving it post-download.

```
---
services:
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - /srv/radarr/config:/config
      - /chungus/media/movies:/movies 
      - /chungus/downloads/rdtclient:/downloads
    ports:
      - 7878:7878
    restart: unless-stopped
```

## Sonarr

Same as Radarr, but for TV/Anime.

```
---
services:
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - /srv/sonarr/config:/config
      - /chungus/media/tv:/tv
      - /chungus/downloads/rdtclient:/downloads
    ports:
      - 8989:8989
    restart: unless-stopped
```

## Prowlarr

This is a torrent indexer and is in charge of actually communicating with torrent sites. It pulls the reports for Radarr and Sonarr to choose from.

```
---
services:
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - /srv/prowlarr/config:/config
    ports:
      - 9696:9696
    restart: unless-stopped
```

## Byparr

This is crucial for communication with a lot of torrent indexers. I find that Byparr has worked the best for me, but there are other options.

```
services:
  byparr:
    image: ghcr.io/thephaseless/byparr:latest
    container_name: byparr
    restart: unless-stopped
    init: true
    ports:
      - "8191:8191"
```

## RDTClient

This is the Real-Debrid downloader. To use this you need to have an account with a Debrid provider, I use real-debrid, it costs around $11 for 90 days of access. [You can sign-up here.](https://real-debrid.com/)

There is one important note to have about configuring the settings for RDTClient to get it to communicate properly with the rest of the stack. The client gives conflicting information to what you should actually use as config.

Under Download Client it says:

> Path in the docker container to download files to (i.e. /data/downloads), or a local path when using as a service.

And under Mapped Path it says:

> Path where files are downloaded to on your host (i.e. D:\Downloads). This path is used for *arr to find your downloads.

One would then assume that you would have these be set to different paths. Wrong, apparently.

The image below is my configuration. Setting them both to /downloads/ does the trick.

<img width="700" height="399" alt="image" src="https://github.com/user-attachments/assets/a9f6dfee-7d98-4818-a4fe-38a1cfaf1ef0" />

This is the part of the install that I was stuck on for a weekend. It was such an easy fix.

```
version: "3.8"

services:
  rdtclient:
    image: rogerfar/rdtclient:latest
    container_name: rdtclient
    restart: unless-stopped
    ports:
      - "6500:6500"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      # rdt-client app data
      - /srv/rdtclient/db:/db

      # downloads (what Sonarr/Radarr will see)
      - /chungus/downloads/rdtclient:/downloads

      # media root (optional but helpful for testing/moves)
      - /chungus/media:/media
```
## Final Notes on this Stack

Those are all of the apps required for minimal functionality. Once you download and get all of them set up, the interaction between the apps is as simple as setting API keys in the right places and mapping where you want your downloads to go. Each applications installation page will have a much better description of this, but for the most part many of them can be set up by just inputting the things that look like they should be input. If you want to add anything else to this server, this is a good baseline.

# Connecting Outside the House

I use [Tailscale](https://tailscale.com/) to connect outside my house. It's incredibly easy to set up by [following the instructions for Linux here](https://tailscale.com/download/linux). This is useful if you're the one able to set it up, but isn't if you're trying to give access to people who aren't as good with technology. It does not require you to open any ports on your network.

I have a seperate dashboard on Homarr with Tailscale IPs to use for connectivity, for the most part I'm just hitting Jellyfin outside the house, and maybe an SSH if I'm feeling fancy.

