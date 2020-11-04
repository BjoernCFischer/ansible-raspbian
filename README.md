# ansible-raspbian

[![Travis (.org)](https://api.travis-ci.com/BjoernCFischer/ansible-raspbian.svg?branch=master)](https://travis-ci.com/github/BjoernCFischer/ansible-raspbian)

This role will setup a secure basic Raspbian environment with sensible defaults. It is forked from [hannseman/ansible-raspbian](https://github.com/hannseman/ansible-raspbian) - thanks and kudos to [Hannes Ljungberg](https://github.com/hannseman).

### It will:

 * Install specified system packages.
 * Configure hostname.
 * Configure network interfaces.
 * Configure locale.
 * Mount tmpfs on write-intensive directories to increase the lifespan of SD-card. 
 * Mount nfs volumes if configured.
 * Change the password on default user.
 * Create additional users.
 * Set the default editor.
 * Setup a secure SSH configuration.
 * Configure UFW.
 * Configure /boot/config.txt.
 * Run raspi-config.
 * Configure Postfix to send email through an SMTP relay.
 * Enable unattended-upgrades.
 * Install Fail2ban.
 * Configure Logwatch to send weekly reports.

### It will not:

 * Update system packages.
 * Run `apt-get update`. Please do this in a pre_task. See [Example Playbook](#example-playbook).
 * Install security patches but unattended-upgrades should take care of that.
 * Remove the default user (pi) 

## Setup
* Install python requirements by running `pip install -r requirements.txt`.
* Install sshpass by running `sudo apt-get install sshpass`.
* Flash SD-card with [Raspbian Stretch  Lite or Buster Lite](https://www.raspberrypi.org/documentation/installation/installing-images/mac.md).
* Add empty file named `ssh` in boot-partition of the flashed SD-card.
* Optional: To enable wifi place a file called `wpa_supplicant.conf` in the boot-partition of the flashed SD-card with the following content:
```
network={
        ssid="your ssid"
        psk="your password"
}
```
* Boot your Raspberry Pi.
* Run playbook.


## Inventory

[sshpass](https://linux.die.net/man/1/sshpass) is required to make the first Ansible run
with the default password `raspberry`. Password authentication over SSH will then be disabled in
preference of public key authentication with keys specified in `ssh_public_keys`. 
Your inventory should contain the following:

```ini
[all:vars]
ansible_connection=ssh
ansible_user=pi
ansible_ssh_pass=raspberry
```

## Variables

```yaml
---

# apt-get installs listed packages
system_packages: []

# The desired system hostname
system_hostname: "raspberrypi"

# The network interface configurations (empty list for "unchanged")
system_interfaces: []
# - {interface: "eth0",
#    ip: "192.168.0.99/24",
#    gateway: "192.168.0.1",
#    dns_servers: "192.168.0.1"}

# The system locale and timezone
system_locale: "en_US.UTF-8"
system_timezone: "Europe/Stockholm"

# List dictionaries of desired tmpfs mounts.
system_tmpfs_mounts:
 - {src: "/run", size: "10%", options: "nodev,noexec,nosuid"}
 - {src: "/tmp", size: "10%", options: "nodev,nosuid"}
 - {src: "/var/log", size: "10%", options: "nodev,noexec,nosuid"}

# List dictionaries of desired nfs mounts.
system_nfs_mounts: []
# - {nfs_share: "192.168.1.22:/my_nfs_share",
#    mountpoint: "/mnt/my_nfs_mount",
#    options: "rw,sync,hard,intr"}

# The password info for ansible_ssh_user (should be configured as pi).
# system_ssh_user_password: the password - CHANGE TO SOMETHING SECURE.
# system_ssh_user_salt: the salt - CHANGE TO SOMETHING SECURE AND RANDOM
system_ssh_user_password: "raspberry"
system_ssh_user_salt: "salt"

# The additional users to create
# username: the desired username
# public_key: the public key for passwordless login
# password: the desired password
# sudoer: flag to indicate the user should be able to sudo commands
# no_sudo_password: flag to indicate the user can sudo passwordless
system_users: []
# - {username: "passwordless_sudoer",
#    public_key: "{{ passwordless_sudoers_pub_key }}",
#    password: 'passwordless_sudoers_password',
#    password_salt: 'salt',
#    sudoer,
#    no_sudo_passwd}
# - {username: "sudoer",
#    public_key: "{{ sudoers_pub_key }}",
#    password: 'sudoers_password',
#    password_salt: 'salt',
#    sudoer}
# - {username: "no_sudoer",
#    public_key: "{{ no_sudoers_pub_key }}",
#    password: 'no_sudoers_password',
#    password_salt: 'salt'}

# Path to default editor
system_default_editor_path: "/usr/bin/vi"

# SSH configurtaion
# ssh_sshd_config: Path to sshd configuration file
# ssh_public_keys: list of ssh public keys to update ~/.authorized_keys.
#                  Note: One of these needs to be one that Ansible is using.
# ssh_banner: String to present when connecting to host over ssh
ssh_sshd_config: "/etc/ssh/sshd_config"
ssh_public_keys: []
ssh_banner: Welcome!

# UFW firewall configuration
# ufw_rules: firewall rules to allow port based communication
#            NOTE: should always allow SSH to keep Ansible functioning
# ufw_allow_igmp: allows/forbids IGMP (multicast) traffic
ufw_rules:
 - {rule: "allow", port: "22", proto: "tcp"}
ufw_allow_igmp: false

# Raspberry Pi configuration
# rpi_boot_config: updates /boot/config.txt with `{{ key }}: {{ value }}`
# rpi_cmdline_config: run raspi-config -noint do_{{ key }} {{ value }].
# See https://github.com/raspberrypi-ui/rc_gui/blob/master/src/rc_gui.c#L23-L70
rpi_boot_config: {}
rpi_cmdline_config: {}

# Postfix configuration
# postfix_hostname: the hostname for outbound mail
# postfix_mailname: the "mailname" - location postfix reads the hostname
# postfix_mydestination: destinations to reach by local transport
# postfix_relayhost: host to relay mail to
# postfix_relayhost_port: relay port (possibly 587)
# postfix_sasl_user: the user on the relay host, e.g. your Gmail address
# postfix_sasl_password: the password on the relay host e.g. your Gmail password
# postfix_smtp_tls_cafile: the trusted CA certs to authenticate relay host
# postfix_rewrite_address: rewrite sender address (if needed by relay host)
postfix_hostname: "{{ ansible_hostname }}"
postfix_mailname: "{{ ansible_hostname }}"
postfix_mydestination:
 - "{{ postfix_hostname }}"
 - localdomain
 - localhost
  - localhost.localdomain
postfix_relayhost: smtp.gmail.com
postfix_relayhost_port: 587
postfix_sasl_user:
postfix_sasl_password:
postfix_smtp_tls_cafile: /etc/ssl/certs/ca-certificates.crt
postfix_rewrite_address:

# Recipient of unattended-upgrades report
unattended_upgrades_email_address: root

# Logwatch configuration.
# logwatch_tmp_dir: logwatch cache directory
# logwatch_mailto: User/email which receives Logwatch reports
# logwatch_detail: Logwatch report detail level
# logwatch_interval: How often to receive Logwatch report (weekly/daily)
logwatch_tmp_dir: /var/cache/logwatch
logwatch_mailto: "root"
logwatch_detail: "Low"
logwatch_interval: "weekly"

# Internal variable used when running tests - should not be used.
ansible_raspbian_testing: false
```

## Example Playbook
```yaml
- hosts: servers
  become: true
  pre_tasks:
    - name: update apt cache
      apt:
        cache_valid_time: 600
  roles:
    - role: hannseman.raspbian
  vars:
    system_packages:
      - apt-transport-https
      - vim
    system_default_editor_path: "/usr/bin/vim.basic"
    system_ssh_user_password: hunter2
    system_ssh_user_salt: pepper
    postfix_sasl_user: root@example.com
    postfix_sasl_password: hunter2

    ssh_public_keys:
      - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJXTGInmtpoG9rYmT/3DpL+0o/sH2shys+NwJLo8NnCj
```
