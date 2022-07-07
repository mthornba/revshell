# GitHub Actions Reverse Shell

In order for the reverse shell to connect to your workstation, you need to
provide a public IP and port (default 2222) forwarded to your local workstation.

## Using netcat

1. Run server in a local terminal:
   ```bash
   nc -vnl 2222
   ```
2. Execute the `Netcat Reverse Shell` workflow

## Using OpenSSL

1. Generate certificate & key:
   ```bash
   openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
   ```
2. Add contents of `cert.pem` to a github secret, `REV_SHELL_CERT`
3. Run server in a local terminal:
   ```bash
   openssl s_server -quiet -key key.pem -cert cert.pem -port 2222
   ```
4. Execute the `OpenSSL Reverse Shell` workflow

### Upgrade the shell to a full TTY

The reverse shell provides a "dumb" terminal that is not a true tty:
```bash
bash-3.2$ openssl s_server -quiet -key key.pem -cert cert.pem -port 2222
bad gethostbyaddr
bash: cannot set terminal process group (679): Inappropriate ioctl for device
bash: no job control in this shell
runner@fv-az249-465:~/work/revshell/revshell$ tty
tty
not a tty
runner@fv-az249-465:~/work/revshell/revshell$ stty
stty
stty: 'standard input': Inappropriate ioctl for device
runner@fv-az249-465:~/work/revshell/revshell$
```

To upgrade the terminal to a full tty:
1. Spawn a pty using,
   `python`
   ```bash
   python -c "import pty;pty.spawn('/bin/bash')"
   ```
   `script`
   ```
   script /dev/null
   ```
2. Put the process in the background and change the local terminal settings
   ```bash
   <Ctrl+Z>
   stty -a
   save_state=$(stty -g)
   stty raw -echo
   ```
3. Return the shell to the foreground and apply tty settings
   ```bash
   fg
   export TERM=xterm-256color
   stty rows ## cols ##
   reset
   ```
4. Do your work in the
stty "$save_state"
```

## Local Testing with `act`

See https://github.com/nektos/act for details on `act`

### netcat

```bash
act -e <(jq --null-input \
  --arg ip "$(ip -4 a | \
  awk '/([0-9]{1,3}\.){3}[0-9]{1,3}/{split($2,ary,/\//);ip=ary[1];if(ip!="127.0.0.1"){print ip;exit}}')" \
  '{ "inputs": { "ip": $ip, "port": 2222 } }') \
-j nc
```

### openssl

```bash
act -s REV_SHELL_CERT="$(<cert.pem)" \
-e <(jq --null-input \
  --arg ip "$(ip -4 a | \
  awk '/([0-9]{1,3}\.){3}[0-9]{1,3}/{split($2,ary,/\//);ip=ary[1];if(ip!="127.0.0.1"){print ip;exit}}')" \
  '{ "inputs": { "ip": $ip, "port": 2222 } }') \
-j openssl
```
After exiting the reverse shell, type `Ctrl-C` to close the openssl server and allow the workflow step to complete.
