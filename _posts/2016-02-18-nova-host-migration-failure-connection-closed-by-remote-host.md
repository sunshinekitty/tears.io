---
layout: post
comments: true
title: Nova host migration failure "Connection closed by remote host"
preview: Recently while trying to migrate a host between nova compute nodes I found that I was getting the following error&#58; `ResizeError&#58; Resize error&#58; not able to execute ssh command&#58; Unexpected error while running command..`
---

Recently while trying to migrate a host between nova compute nodes I found that I was getting the following error: `ResizeError: Resize error: not able to execute ssh command: Unexpected error while running command..` As it turns out, once you understand what Nova is trying to do this is pretty easy to fix. Each nova user on each node will be trying to log in to the other server, and so the nova user on each node will need to have the other’s ssh public key installed. It also helps to disable strict host key checking.

On both nodes:

```bash
usermod -s /bin/bash nova  
su - nova  
ssh-keygen -t rsa  
# don’t set a password
cat << EOF > ~/.ssh/config  
Host *  
    StrictHostKeyChecking no
    UserKnownHostsFile=/dev/null
EOF 
```

Next take the public key from each hypervisor and add it to the other. This can be done by opening `~/.ssh/id_rsa.pub`, copying the key, and adding it to the other servers `~/.ssh/authorized_keys` file.

Congratulations, you should be set.
