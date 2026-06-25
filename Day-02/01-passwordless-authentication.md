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



### Deep Dive

In your example, the **PEM file** is the AWS private key file used to prove you are allowed to log in to an EC2 instance.

Example:

```bash
~/keys/controlNode.pem
```

This is a **private key**. It must stay secret.

AWS put the matching **public key** on the EC2 instance when the instance was created. So SSH works like this:

```text
Your private key: ~/keys/controlNode.pem
Remote instance has matching public key: ~/.ssh/authorized_keys
```

In your command:

```bash
ssh-copy-id -f "-o IdentityFile ~/keys/controlNode.pem" ubuntu@3.27.138.77
```

SSH does this:

```text
Master node uses ~/keys/controlNode.pem to log in to 3.27.138.77
```

If `3.27.138.77` accepts that PEM, login succeeds.

Then `ssh-copy-id` copies the master node’s public key, usually:

```bash
/home/ubuntu/.ssh/id_ed25519.pub
```

into the agent node:

```bash
/home/ubuntu/.ssh/authorized_keys
```

After that, the master node can connect to the agent node using its own key pair:

```text
Master private key:
 /home/ubuntu/.ssh/id_ed25519

Agent authorized_keys contains:
 /home/ubuntu/.ssh/id_ed25519.pub
```

So the flow is:

```text
Before ssh-copy-id:

Master node
  has PEM file: ~/keys/controlNode.pem
  has Ansible SSH key: ~/.ssh/id_ed25519 and ~/.ssh/id_ed25519.pub

Agent node 3.27.138.77
  accepts AWS PEM's matching public key

Master connects to Agent using PEM:
  ssh -i ~/keys/controlNode.pem ubuntu@3.27.138.77


During ssh-copy-id:

Master logs into Agent using PEM
Master copies ~/.ssh/id_ed25519.pub
Agent saves it into ~/.ssh/authorized_keys


After ssh-copy-id:

Master can connect to Agent without PEM:
  ssh ubuntu@3.27.138.77
```

Simple difference:

```text
PEM file
```

Temporary AWS access key used to enter the EC2 instance.

```text
~/.ssh/id_ed25519
```

Master node’s own private key used later by Ansible.

```text
~/.ssh/id_ed25519.pub
```

Master node’s public key copied to agent.

```text
agent ~/.ssh/authorized_keys
```

List of public keys allowed to log in to the agent.

For Ansible, final desired connection is:

```bash
master node ---> agent node
```

using:

```bash
ssh ubuntu@3.27.138.77
```

without manually giving:

```bash
-i ~/keys/controlNode.pem
```


### SUMMARY
Short flow:

```text
1. PEM file is AWS private key.
   It is used to log in to an EC2 instance initially.

2. Master node uses PEM to connect to agent node once:
   ssh -i ~/keys/controlNode.pem ubuntu@3.27.138.77

3. ssh-copy-id uses that PEM to log in,
   then copies master node's public key to agent node.

4. The copied key goes here on agent:
   /home/ubuntu/.ssh/authorized_keys

5. After that, master can SSH to agent without PEM:
   ssh ubuntu@3.27.138.77

6. Then Ansible can also connect from master to agent passwordlessly.
```

Command:

```bash
ssh-copy-id -f -i ~/.ssh/id_ed25519.pub -o IdentityFile=~/keys/controlNode.pem ubuntu@3.27.138.77
```

Meaning:

```text
Use PEM to enter agent once,
copy master's public key to agent,
then future SSH uses master's normal SSH key.
```
