# Prerequisites

In this lab you will review the machine requirements necessary to follow this tutorial.

## Virtual or Physical Machines

This tutorial requires four (4) virtual or physical ARM64 or AMD64 machines running Debian 12 (bookworm). The following table lists the four machines and their CPU, memory, and storage requirements.

| Name    | Description            | CPU | RAM   | Storage |
|---------|------------------------|-----|-------|---------|
| jumpbox | Administration host    | 1   | 512MB | 10GB    |
| server  | Kubernetes server      | 1   | 2GB   | 20GB    |
| node-0  | Kubernetes worker node | 1   | 2GB   | 20GB    |
| node-1  | Kubernetes worker node | 1   | 2GB   | 20GB    |

How you provision the machines is up to you, the only requirement is that each machine meet the above system requirements including the machine specs and OS version.

### Vagrant setup
Hereby, we'll do the provisioning with [Vagrant](https://developer.hashicorp.com/vagrant) and [VirtualBox](https://www.virtualbox.org/). 

1. [Install Vagrant](https://developer.hashicorp.com/vagrant/install) and [install VirtualBox](www.virtualbox.org/wiki/Downloads).
2. Go to the root of the repo and do ```vagrant up``` to bring up the 4 machines. This can take a few minutes.
3. When it's done:
    ```bash
    $ vagrant status
    Current machine states:

    jumpbox                   running (virtualbox)
    server                    running (virtualbox)
    node-0                    running (virtualbox)
    node-1                    running (virtualbox)
    ```
4. Find their IPs - `eth1` is the private network interface (eth0 is Vagrant's NAT). Note all 4 IPs.
    ```bash
    $ vagrant ssh jumpbox -- ip addr show eth1
    $ vagrant ssh server  -- ip addr show eth1
    $ vagrant ssh node-0  -- ip addr show eth1
    $ vagrant ssh node-1  -- ip addr show eth1
    ```
5. Verify that they can talk to each other. E.g. check that `jumpbox` can talk to the other 3:
    ```bash
    $ vagrant ssh jumpbox -- ping -c2 <server-ip>
    $ vagrant ssh jumpbox -- ping -c2 <node-0-ip>
    $ vagrant ssh jumpbox -- ping -c2 <node-1-ip>
    ```

Once you have all four machines provisioned, verify the OS requirements by viewing the `/etc/os-release` file:

6. Set-up SSH from `jumpbox` to the others: 
    ```bash
    # dump the SSH config
    $ vagrant ssh-config > /tmp/vagrant-ssh.cfg

    # copy server private key to jumpbox
    $ scp -F /tmp/vagrant-ssh.cfg \
    .vagrant/machines/server/virtualbox/private_key \
    jumpbox:/home/vagrant/.ssh/server_key

    $ scp -F /tmp/vagrant-ssh.cfg \
    .vagrant/machines/node-0/virtualbox/private_key \
    jumpbox:/home/vagrant/.ssh/node0_key

    $ scp -F /tmp/vagrant-ssh.cfg \
    .vagrant/machines/node-1/virtualbox/private_key \
    jumpbox:/home/vagrant/.ssh/node1_key

    # fix permissions
    $ vagrant ssh jumpbox -- chmod 600 /home/vagrant/.ssh/server_key /home/vagrant/.ssh/node0_key /home/vagrant/.ssh/node1_key
    ```

7. Create an SSH config file on `jumphost`:
`vagrant ssh jumpbox` and then e.g. `nano ~/.ssh/config` with these contents:
    ```bash
    Host server
    HostName <server-ip>
    User vagrant
    IdentityFile ~/.ssh/server_key

    Host node-0
    HostName <node-0-ip>
    User vagrant
    IdentityFile ~/.ssh/node0_key

    Host node-1
    HostName <node-1-ip>
    User vagrant
    IdentityFile ~/.ssh/node1_key
    ```

8. Verify that from within `jumpbox` you can do e.g. `ssh node-0`

### Enabling root access

1. Permit root logins on all 4 boxes:
    ```bash
    for vm in jumpbox server node-0 node-1; do
    vagrant ssh $vm -- sudo bash -c \
        "'echo root:root | chpasswd && \
        sed -i s/^#PermitRootLogin.*/PermitRootLogin\ yes/ /etc/ssh/sshd_config && \
        sed -i s/^PermitRootLogin.*/PermitRootLogin\ yes/ /etc/ssh/sshd_config && \
        systemctl restart sshd'"
    done
    ```
2. Get the IPs of all 4 boxes:
    ```bash
    for vm in jumpbox server node-0 node-1; do
        echo -n "$vm: "
        vagrant ssh $vm -- ip addr show eth1 | grep "inet " | awk '{print $2}' | cut -d/ -f1
    done
    ```

3. Add the IPs to your `/etc/hosts`:

    ```bash
    sudo tee -a /etc/hosts <<EOF
    192.168.x.x  jumpbox
    192.168.x.x  server
    192.168.x.x  node-0
    192.168.x.x  node-1
    EOF
    ```

4. Also add them on the jumpbox:

    ```bash
    vagrant ssh jumpbox -- sudo tee -a /etc/hosts <<EOF
    192.168.x.x  server
    192.168.x.x  node-0
    192.168.x.x  node-1
    EOF
    ```

5. Set-up key-based authentication:
    ```bash
    # Generate a key on your host if you don't have one
    ssh-keygen -t ed25519 -f ~/.ssh/k8s-hard-way -N ""

    # Copy it to all 4 VMs as root
    for vm in jumpbox server node-0 node-1; do
    vagrant ssh $vm -- sudo bash -c \
        "'mkdir -p /root/.ssh && cat >> /root/.ssh/authorized_keys'" \
        < ~/.ssh/k8s-hard-way.pub
    done
    ```

6. Finally, add this to your host's `~/.ssh/config`:
    ```bash
    Host jumpbox server node-0 node-1
    User root
    IdentityFile ~/.ssh/k8s-hard-way

    ```

## Verifications
From the host machine: 

```bash
ssh root@jumpbox
```

Then, when inside:
```bash
cat /etc/os-release
```

You should see something similar to the following output:

```text
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
NAME="Debian GNU/Linux"
VERSION_ID="12"
VERSION="12 (bookworm)"
VERSION_CODENAME=bookworm
ID=debian
```

Next: [setting-up-the-jumpbox](02-jumpbox.md)
