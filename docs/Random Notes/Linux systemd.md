Is the first process that get initiated after starting a linux kernel, almost all distros have it nowadays (not `init.d` anymore.) It starts its services on startup, eases process management by starting/stopping them, and is integrated with `journalctl` for logging. A typical `/etc/systemd/system/tomcat@service` tomcat file would look like this:
```bash
[Unit]
Description=Tomcat - instance %i
After=syslog.target network.target

[Service]
Type=forking

User=tomcat
Group=tomcat

WorkingDirectory=/var/tomcat/%i

Environment="JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/"
Environment="JAVA_OPTS=-Djava.security.egd=file:///dev/urandom"

Environment="CATALINA_PID=/var/tomcat/%i/run/tomcat.pid"
Environment="CATALINA_BASE=/var/tomcat/%i/"
Environment="CATALINA_HOME=/opt/tomcat/"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

#RestartSec=10
#Restart=always

[Install]
WantedBy=multi-user.target
```
then it can be controlled by:
```bash
sudo systemctl enable tomcat   # Start at boot
sudo systemctl start tomcat
sudo systemctl status tomcat
```

## Docker `supervisord`
The problem with docker is, that it is not a full virtual machine, instead it only uses isolated processes. That means that `systemd` is not available to docker.

If you want to run multiple processes in a docker image like `tomcat` and `cron` at the same time, problems arise, as `systemd` does not exist. `supervisord` enables docker images to host multiple processes:
```bash
[program:tomcat]
command=/opt/tomcat/bin/catalina.sh run
user=tomcat
environment=HOME="/opt/tomcat"
environment=USER="tomcat"
autorestart=true
startsecs=2
stdout_logfile=/var/log/tomcat.log
priority=890
  
[program:cron]
command=cron -f
startsecs=2
autorestart=true
```