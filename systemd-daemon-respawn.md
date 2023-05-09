Quick recipe to manage processes with `systemd`, specifically to restart a dead daemon.

- Write the script you want systemd to manage `/usr/bin/connector-launch.sh`
```
#!/bin/bash

mongosqld --mongo-uri "mongodb://moo.mongodb.net:27017/?retryWrites=true&w=majority" --auth -u root -p foo --addr 0.0.0.0:3309 --mongo-ssl --sslMode requireSSL --sslPEMKeyFile /usr/bin/mongo.pem
```

- Write a systemd unit file `/etc/systemd/system/script-manager.service`

```
[Unit]
Description=mysqld

[Service]
ExecStart=/usr/bin/connector-launch.sh
Restart=on-failure
```

- Reload the systemd manager configuration with the command: `systemctl daemon-reload`
- start the service `systemctl start myhttpservice.service`
- check service status `systemctl status myhttpservice.service`

Hopefully one day you get around to figuring out why the daemon was dying in the first place. Hopefully.
