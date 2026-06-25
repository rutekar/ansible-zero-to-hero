# How to setup Passwordless Authentication

## EC2 Instances

### Using Public Key

```
ssh-copy-id -f "-o IdentityFile <PATH TO PEM FILE>" ubuntu@<INSTANCE-PUBLIC-IP>
```

- ssh-copy-id: This is the command used to copy your public key to a remote machine.
- -f: This flag forces the copying of keys, which can be useful if you have keys already set up and want to overwrite them.
- "-o IdentityFile <PATH TO PEM FILE>": This option specifies the identity file (private key) to use for the connection. The -o flag passes this option to the underlying ssh command.
- ubuntu@<INSTANCE-IP>: This is the username (ubuntu) and the IP address of the remote server you want to access.

### Using Password 

- Go to the file `/etc/ssh/sshd_config.d/60-cloudimg-settings.conf`
- Update `PasswordAuthentication yes`
- Restart SSH -> `sudo systemctl restart ssh`

### To Connect master node to to gent node using passwordless ssh keys
- Login to master node using master node PEM file (pem file should be permission chmod 400)
- copy pem file of both master node and agent node on master node (here both agent node has same PEM file and master node has different PEM file)
  using below command from Downloads folder where PEM files downloaded
  - For master server key
    ```
    scp -i "masterNode.pem" masterNode.pem  ubuntu@ec2-3-25-163-73.ap-southeast-.compute.amazonaws.com:/home/ubuntu/keys
    ```   (create keys folder on master server)

  - For agent node key file copy to master node
    `scp -i "masterNode.pem" controlNode.pem  ubuntu@ec2-3-25-163-73.ap-southeast-.compute.amazonaws.com:/home/ubuntu/keys`

 - Once PEM files copied,change to PEM file permissions to chmod 600
 - Then test connection to agent node using agent node PEM key and agent node public ip
   `ssh -i ~/keys/controlNode.pem ubuntu@3.27.138.77`
   press exit to back to master node
 - Once connection success, genarate public and private on master node using below command which public SSH key will later move to agent server to have password less authentication
   `ssh-keygen`
 - Then run below command which will move generated above public SSH key to agent node 
   `ssh-copy-id -f "-o IdentityFile ~/keys/controlNode.pem" ubuntu@3.27.138.77`
 - Then the same will allow master node to connect agent directly using ssh command
   `ssh ubuntu@3.27.138.77`
 - After move public SSH key to agent node we can check the same in authorized_keys file on agent node
   `cat ~/.ssh/authorized_keys`
 - Do the same for other agent server also.
