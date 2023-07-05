---
title: "OpenTTD hosting"
date: 2023-06-30T18:29:39+02:00
tags: ["linux", "openttd", "gry", "hostowanie", "techniczne"]
authors: ["Bartłomiej Kudzia"]
---

## OpenTTD Installation

```bash
# As root, from the /root directory
curl -O https://cdn.openttd.org/openttd-releases/13.0/openttd-13.0-linux-generic-amd64.tar.xz
curl -O https://cdn.openttd.org/opengfx-releases/7.1/opengfx-7.1-all.zip
unzip opengfx-7.1-all.zip
cd /opt
tar xf /root/openttd-13.0-linux-generic-amd64.tar.xz
mv openttd-13.0-linux-generic-amd64 openttd
cp /root/opengfx-7.1.tar /opt/openttd/baseset
```

## User Creation

OpenTTD stores add-ons, saves, etc. in the user's folder, and running the server as the root user is not a good idea.

```bash
useradd -m -d /srv/openttd -s /bin/bash openttd
```

## First Run

During the first run, OpenTTD will generate the `~/.config/openttd` and `~/.local/share/openttd` folders.

```bash
su openttd
/opt/openttd/openttd -D
```

After starting the server, you can immediately configure some settings.

```bash
server_name FernTTD
name FernTTD
rcon_pw j3b4cp1s
setting server_game_type public
```

## Scenario and Add-ons Installation

Place the `init.sav` file in the `/srv/openttd/.local/share/openttd/save` directory. Remember to change the permissions.

```bash
chown openttd:openttd /srv/openttd/.local/share/openttd/save/init.sav
```

This file contains the initial game state, so you need to make a copy of it. Make sure to execute the commands as the `openttd` user!

```bash
cp /srv/openttd/.local/share/openttd/save/init.sav /srv/openttd/.local/share/openttd/save/main.sav
```

Upload the `newgrf.tar.xz` file to the `/srv/openttd` directory and extract it to the `.local/share/openttd/newgrf` folder:

```bash
cd ~/.local/share/openttd/newgrf
tar xvf ~/newgrf.tar.xz
```

Now, you need to start the server and install the required add-ons.

```bash
/opt/openttd/openttd -D
```

```bash
rescan_newgrf
content update
content select 16509003
content download
```

To check if everything is working, you can try running the server with the save file:

```bash
/opt/openttd/openttd -D -g save/main.sav
```

## Scripts

In the home directory of the `openttd` user, create a script called `launch_openttd.sh` and paste the following code:

```bash
#!/bin/bash

# Change to /srv/openttd
cd

save_dir=.local/share/openttd/save

# If main.sav doesn't exist, make a copy of init.sav
[[ -f $save_dir/main.sav ]] || cp -v $save_dir/init.sav $save_dir/save/main.sav

recent_autosave=`ls -t $save_dir/autosave | head -n1`

# If there's an autosave
if ! [[ -z $recent_autosave ]]
then
        # If the autosave is newer than main.sav, replace it
        [[ $save_dir/autosave/$recent_autosave -nt $save_dir/main.sav ]] \
                && cp -v $save_dir/autosave/$recent_autosave $save_dir/main.sav
fi

# Start the server
/opt/openttd/openttd -D -g save/main.sav >>server.log
```

Create another script called `restart_game.sh`:

```bash
#!/bin/bash
cp -v $save_dir/init.sav $save_dir/save/main.sav
```

The first script copies the latest autosave to the `main.sav` file and starts the server. The second script removes the `main.sav` file so that it will be created again when the server starts. Make sure to set the appropriate permissions:

```bash
chmod u+x launch_openttd.sh
chmod u+x restart_game.sh
```

## Service Configuration

Create a file named `/etc/systemd/system/openttd.service` and paste the following content:

```cfg
[Unit]
Description=OpenTTD Dedicated Server
After=local-fs.target network.target

[Service]
WorkingDirectory=/srv/openttd
User=openttd
Group=openttd
ExecStart=/bin/bash /srv/openttd/launch_openttd.sh
Type=simple

[Install]
WantedBy=multi-user.target
```

Then refresh systemd:

```bash
systemctl daemon-reload
```

This allows you to manage the server using standard commands:

```
systemctl start openttd
systemctl stop openttd
systemctl enable openttd
systemctl disable openttd
systemctl status openttd
```

## Ports

OpenTTD uses ports `3978` & `3979` (TCP and UDP).

## Joining the Game

After connecting, if the game is paused, you can unpause it using the command: `rcon j3b4cp1s unpause`
