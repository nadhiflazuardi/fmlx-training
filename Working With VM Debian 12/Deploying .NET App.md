Publish .NET project
```
dotnet publish -c Release -r linux-arm64 --self-contained false -o ./publish
```

Copy publish folder to Raspberry Pi

On the Raspberry Pi, install .NET
`wget https://packages.microsoft.com/config/debian/12/packages-microsoft-prod.deb -O packages-microsoft-prod.deb`
`sudo dpkg -i packages-microsoft-prod.deb`
`rm packages-microsoft-prod.deb`
`sudo apt update`
`sudo apt install -y dotnet-runtime-9.0`

From the Raspberry Pi, try running the executable file
`dotnet run <executable file name>`

# Making a Service to Start App On Boot
---
Make a service file with `path /etc/systemd/system/<service-name>.service`. Below is an example
```
[Unit]
Description=Weather App API
After=network.target

[Service]
Type=notify
ExecStart=/home/nadhif/TestWeatherApp2/WebApplication1
Restart=always
User=nadhif

[Install]
WantedBy=multi-user.target
```

Reload systemd
```bash
sudo systemctl daemon-reload
```

Enable service to start on boot
```bash
sudo systemctl enable <service-name>
```

To start the service now,
```bash
sudo systemctl start <service-name>
```

### Checking Logs
Outputs and errors from the service can be seen through `journalctl`
```bash
journalctl -u <service-name>
```

### Stopping or Disabling a Service
To stop now (will be restarted on boot)
```bash
sudo systemctl stop <service-name>
```

To disable the service (will no longer started on boot)
```bash
sudo systemctl disable <service-name>
```