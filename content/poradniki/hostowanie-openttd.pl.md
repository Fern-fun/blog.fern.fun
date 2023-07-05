---
title: "Hostowanie OpenTTD"
date: 2023-06-30T18:29:39+02:00
tags: ["linux", "openttd", "gry", "hostowanie", "techniczne"]
authors: ["bartek"]
---

## Instalacja OpenTTD
```bash
# Jako root, z katalogu /root
curl -O https://cdn.openttd.org/openttd-releases/13.0/openttd-13.0-linux-generic-amd64.tar.xz
curl -O https://cdn.openttd.org/opengfx-releases/7.1/opengfx-7.1-all.zip
unzip opengfx-7.1-all.zip
cd /opt
tar xf /root/openttd-13.0-linux-generic-amd64.tar.xz
mv openttd-13.0-linux-generic-amd64 openttd
cp /root/opengfx-7.1.tar /opt/openttd/baseset
```
## Tworzenie użytkownika
OpenTTD przechowuje dodatki, savey itp. w folderze użytkownika, a uruchamianie serwera jako użytkownik root nie jest dobrym pomysłem.
```bash
useradd -m -d /srv/openttd -s /bin/bash openttd
```
## Pierwsze uruchomienie
Przy pierwszym uruchomieniu OpenTTD wygeneruje foldery `~/.config/openttd` i `~/.local/share/openttd`
```bash
su openttd
/opt/openttd/openttd -D
```
Po uruchomieniu odpaleniu serwera można od razu skonfigurować część ustawień
```bash
server_name FernTTD
name FernTTD
rcon_pw j3b4cp1s
setting server_game_type public
```
## Instalacja scenariusza i dodatków
Wrzuć plik `init.sav` do katalogu `/srv/openttd/.local/share/openttd/save`. Pamiętaj o zmianie uprawnień.
```bash
chown openttd:openttd /srv/openttd/.local/share/openttd/save/init.sav
```
Ten plik zawiera początkowy stan gry, należy go skopiować do nowego pliku. Upewnij się, że polecenia wykonujesz jako użytkownik `openttd`!
```bash
cp /srv/openttd/.local/share/openttd/save/init.sav /srv/openttd/.local/share/openttd/save/main.sav
```
Wrzuć na serwer plik `newgrf.tar.xz` do katalogu `/srv/openttd` i rozpakuj do folderu `.local/share/openttd/newgrf`:
```bash
cd ~/.local/share/openttd/newgrf
tar xvf ~/newgrf.tar.xz
```
Teraz należy odpalić serwer i zainstalować potrzebne dodatki
```bash
/opt/openttd/openttd -D
```
```bash
rescan_newgrf
content update
content select 16509003
content download
```
Aby sprawdzić, czy wszystko działa, można spróbować uruchomić serwer z zapisu:
```bash
/opt/openttd/openttd -D -g save/main.sav
```
## Skrypty
W katalogu domowym użytkownika `openttd` utwórz skrypt `launch_openttd.sh` i wklej ten kod:
```bash
#!/bin/bash

# przejdz do /srv/openttd
cd

save_dir=.local/share/openttd/save

# jesli main.sav nie istnieje, utworz kopie init.sav
[[ -f $save_dir/main.sav ]] || cp -v $save_dir/init.sav $save_dir/save/main.sav

recent_autosave=`ls -t $save_dir/autosave | head -n1`

# jesli istnieje autozapis
if ! [[ -z $recent_autosave ]]
then
        # jesli autozapis jest nowszy od main.sav, to go podmien
        [[ $save_dir/autosave/$recent_autosave -nt $save_dir/main.sav ]] \
                && cp -v $save_dir/autosave/$recent_autosave $save_dir/main.sav
fi

# uruchom serwer
/opt/openttd/openttd -D -g save/main.sav >>server.log
```
Utwórz jeszcze drugi skrypt `restart_game.sh`:
```bash
#!/bin/bash
cp -v $save_dir/init.sav $save_dir/save/main.sav
```
Pierwszy skrypt kopiuje najnowszy autozapis do pliku `main.sav` i włącza serwer, a drugi usuwa plik `main.sav`, więc zostanie on utworzony przy starcie serwera.
Pamiętaj o ustawieniu odpowiednich uprawnień.
```bash
chmod u+x launch_openttd.sh
chmod u+x restart_game.sh
```
## Konfiguracja usługi
Utwórz plik `/etc/systemd/system/openttd.service` i wklej zawartość
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
Następnie odśwież systemd
```bash
systemctl daemon-reload
```
Dzięki temu można zarządzać serwerem używając standardowych komend
```
systemctl start openttd
systemctl stop openttd
systemctl enable openttd
systemctl disable openttd
systemctl status openttd
```
## Porty
OpenTTD korzysta z portów `3978` i `3979` (TCP i UDP).
## Dołączenie do gry
Gdy po połączeniu gra będzie zapauzowana, można ją odpauzować tą komendą: `rcon j3b4cp1s unpause`.
