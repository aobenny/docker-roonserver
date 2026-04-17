# docker-roonserver

**April 2026: DEPRECATION NOTICE**

Roonlabs finally launched it's own Docker image to run Roon inside a Docker container.
It can be found [here](https://github.com/RoonLabs/roon-docker).

I strongly suggest all users of my image to migrate to the official image from Roonlabs. There will be no more updates of my image, and I will take it offline later this year.

## Migration from my image to Roonlabs image

On the github link above you can fill out a form which generates `docker-compose.yaml` or a shell command to start the container.

You can reuse existing host volumes for music and backups.

You can *not* reuse the `roon-app` and `roon-data` volumes, because they were separate in my image, but are combined in the image from Roonlabs.
So please specify a *brand-new* docker volume for the `/Roon` folder in the above mentioned form.

After that, you can copy the contents of your old `roon-data` volume to this new volume, in the command below mentioned as `roon-data-new`:

    sudo docker run --rm -ti -v roon-data:/data -v roon-data-new:/new alpine cp -pr /data /new/

Please doublecheck other commandline options you may have in place and the commandline options that are suggested by Roonlabs.

Finally, cross fingers and start the Roon container with the new settings.
If all went well your phone app and other clients and players should be connected to the new Roon container.
All your sources, playlists etc should still be there.

Happy listening!

Steef



# Old README

RoonServer downloading Roon on first run

This little project configures a docker image for running RoonServer.
It downloads RoonServer if not found on an external volume.

Example start:

    docker run -d \
      --net=host \
      -e TZ="Europe/Amsterdam" \
      -v roon-app:/app \
      -v roon-data:/data \
      -v roon-music:/music \
      -v roon-backups:/backup \
      steefdebruijn/docker-roonserver:latest
  
  * You should set `TZ` to your timezone.
  * You can change the volume mappings to local file system paths if you like.
  * You *must* use different folders for `/app` and `/data`.
    The app will not start if they both point to the same folder or volume on your host.
  * You should set up your library root to `/music` and configure backups to `/backup` on first run.

## Systemd

If you deploy on a host with `systemd`, you should use a systemd service to start the Roon service.

Example `systemd` service (adapt to your environment):

    [Unit]
    Description=Roon
    After=docker.service
    Requires=docker.service
    
    [Service]
    TimeoutStartSec=0
    TimeoutStopSec=180
    ExecStartPre=-/usr/bin/docker kill %n
    ExecStartPre=-/usr/bin/docker rm -f %n
    ExecStartPre=/usr/bin/docker pull steefdebruijn/docker-roonserver
    ExecStart=/usr/bin/docker \
      run --name %n \
      --net=host \
      -e TZ="Europe/Amsterdam" \
      -v roon-app:/app \
      -v roon-data:/data \
      -v roon-music:/music \
      -v roon-backups:/backup \
      steefdebruijn/docker-roonserver
    ExecStop=/usr/bin/docker stop %n
    Restart=always
    RestartSec=10s
    
    [Install]
    WantedBy=multi-user.target

## Docker-compose

If you deploy in a `docker-compose` environment, create a `docker-compose.yaml` file and run `docker-compose run <service>`.

Example `docker-compose.yaml` (adapt to your environment):

    version: "3.7"
    services:
      docker-roonserver:
        image: steefdebruijn/docker-roonserver:latest
        container_name: docker-roonserver
        hostname: docker-roonserver
        network_mode: host
        environment:
          TZ: "Europe/Amsterdam"
        volumes:
          - roon-app:/app
          - roon-data:/data
          - roon-music:/music
          - roon-backups:/backup
        restart: always
    volumes:
      roon-app:
      roon-data:
      roon-music:
      roon-backups:


## Network shares

If you find yourself in trouble using remote SMB/CIFS shares, you probably need some additional privileges on the container.
You have two options here (see also issue #15):

Run the Roon container in privileged mode

    # standalone or from systemd service:
    docker run --privileged --name roonserver ...

    # docker-compose.yaml (inside service section):
    privileged: true

Run the Roon container with the right privileges. Some of these are docker-related, but depending on your host distribution and security settings you may need additional privileges.

    # standalone or from systemd service:
    docker run --cap-add SYS_ADMIN --cap-add DAC_READ_SEARCH --security-opt apparmor:unconfined ...
    
    # docker-compose.yaml (inside service section):
    cap_add:
      - SYS_ADMIN
      - DAC_READ_SEARCH
    security_opt:
      - apparmor:unconfined


## Network issues

  If your docker host has multiple networks attached and your core has trouble finding audio sinks/endpoints, you can try using a specific docker network setup as described in issue #1:

    docker network create -d macvlan \
       --subnet 192.168.1.0/24 --gateway 192.168.1.1 \
       --ip-range 192.168.1.240/28 -o parent=enp4s0 roon-lan
    docker run --network roon-lan --name roonserver ...

  Use the subnet and corresponding gateway that your audio sinks/endpoints are connected to. Use an ip-range for docker that is not conflicting with other devices on your network and outside of the DHCP range on that subnet if applicable.

## Extensions

If you would like to use the Roon extensions, please deploy a separate docker container for the extension manager, for example [this one](https://hub.docker.com/r/theappgineer/roon-extension-manager).
I have not tried this myself, I do not use Roon extensions.

## Sound cards on host

If you want to use soundcard(s) attached to docker host, try mounting the soundcard device(s) to the roon container.

Direct or via systemd:

    docker run --device /dev/snd:/dev/snd ....

or via compose:

    devices:
      - "/dev/snd:/dev/snd"

See also issue #26.

## Backups

  Don't forget to backup the `roon-backups` *for real* (offsite preferably).

  Have fun!
  
  Steef

## Version history

  * 2026-04-17: Deprecation notice and migration guide towards official Roonlabs image
  * 2025-07-11: base image is still 'debian:12-slim', as it seems to be the latest available. Image is recompiled for latest debian updates.
  * 2023-11-03: update base image to 'debian:12-slim', dependency to libicu72.
  * 2022-04-12: update base image to 'debian:11-slim'.
  * 2022-03-19: Fix download URL, follow redirects on download. Added specific usage scenarios in README.
  * 2021-05-24: update base image to `debian:10.9-slim` and check for shared `/app` and `/data` folders.
  * 2019-03-18: Fix example start (thanx @heapxor); add `systemd` example.
  * 2019-01-23: updated base image to `debian-9.6`
  * 2017-08-08: created initial images based on discussion on roonlabs forum.


