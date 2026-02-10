# chungus-box


# Hardware

ChungusBox is running on a refurbished Dell Optiplex 3050 SFF. These can be found for anywhere from $50-$200 dollars, at varying hardware levels. The main thing is getting a 7th Gen or newer Intel Processor.

My specs are:

- Intel Core i5-7500 @ 3.8 GHz
- 16GB DDR4 @ 2133 MHz
- No-name 256GB SSD for boot
- Seagate 8TB HDD

In the future the goal is to have this machine serve as a brain hooked up to a NAS, for now it serves it's purpose.

# Operating System

I used Ubuntu Desktop 24.04.3, if I was going to do it again I would probably just run Ubuntu server, but desktop is helpful for first time setup if you aren't comfortable in Linux terminal. After original setup, I've primarily interacted with it via SSH and Web Apps.

You could theoretically run this on Windows but the entire workflow would be different and you'd be on your own.

# Setting up the Basics

I am not someone who would claim to have an Linux knowledge. After putting this together, I can confidently download things that work on the first try, and solidly navigate the Linux file structure via terminal. Hopefully, those are also the experiences you gain by following along. I'm also vaguely remembering the steps I took at this part. I trust that you, dear reader, can run basic Linux cleanup and lookup how to install something if it says it's not installed.

## Static IP

First thing has nothing to do with Linux. Give your server a DHCP reservation to keep a static IP. This is different depending on your ISP/router, but all of them can do it. I gave mine the IP of 192.168.1.67 because I'm horribly brain-rotted and this machine is called ChungusBox.

If you don't know how to find the MAC address:


```
ip link
```


Technically, you should keep it as whatever it's set as but I don't care haha 6-7 funny.

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

3. Check that it worked.


```
sudo docker run hello-world
```


### Portainer

This will be your first browser app and is used as a visualization tool for Docker. I can't use Docker in the terminal I'm too lazy to figure out how. Here's how to install Portainer.

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

4. Do your first time set-up!

# The Everything Else Section

Pretty much everything from this point on is just the Docker Compose files. You can access the section to add these in Portainer by going to your Local Environments > Stacks > Add Stack. Here you can paste your Docker Compose files.

Below is going to be direct rips of my Docker Composes. Pretty much everything can be kept the same, but specifically your volume mappings may have to be different. If you wanna copy mine, that's totally fine.

All of my config files live in the /srv/ directory within a folder for their specific app, example below.


```
/srv/APP_NAME/config
```

The rest of the everything lives on an 8TB Seagate that is mounted to /chungus/, aptly named. Here is the basic file structure for that. The easiest way to follow along would be to map your drive to something and work off that folder directly. It doesn't have to be /chungus/, but if you change it to /boomerang/, you only have to change the instances of chungus, not everything entirely.

Same with /srv/ but your config files can kind of live wherever you want them too, I don't care, and it isn't super duper important, or it is and I don't know enough to say otherwise.


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

The pipeline can basically be boiled down to this.

```
Downloaded to /chungus/downloads --> Moved by Radarr/Sonarr to correct /chungus/media/... --> Picked up by Jellyfin
```


## Minimum Functionality

These are apps required for the one click download workflow. Simply copy and paste these Docker Composes where they belong and deploy the stack. Each app can be accessed via the static-ip you set with the ports set within the file under ports.

Pay attention to the volume mapping, I'd recommend reading into it more, but in short the left side is your computer, the right side is what docker sees. It's important to have many of these mapped to similar locations, like radarr and rdt-client.

### Jellyfin (Media Viewing)


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

### Jellyseerr (Media Request)


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

### Radarr (Movies)


