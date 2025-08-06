### Generate SSH Key Pair
```bash
ssh-keygen -t rsa -b 4096
```
- Press Enter to accept the default file location (`~/.ssh/id_rsa`)

### Copy Public Key to Remote Server
```bash
ssh-copy-id username@remote_server
```

### Set Permission on Remote Server
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### Try Connecting
```bash
ssh username@remote_server
```
