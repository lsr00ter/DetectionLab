# Building DetectionLab on ESXi

Documentation: <https://www.detectionlab.network/deployment/esxi/>

An additional guide for ESXi deployment can be found at <https://clo.ng/blog/detectionlab-on-esxi/>

<!-- [![Overview](../img/ESXi%20%20DetectionLab/esxi_overview.jpeg)](https://github.com/clong/DetectionLab/blob/master/img/esxi_overview.jpeg?raw=true&width=1200) -->

An additional step-by-step guide can be found here which also details the ESXi installation process: [https://clo.ng/blog/detectionlab-on-esxi/](https://clo.ng/blog/detectionlab-on-esxi/)

## Prereqs (~30-60 minutes)

1. Have an ESXi instance version 6 or higher. VSphere is **NOT** required.
2. [Terraform](https://developer.hashicorp.com/terraform/install) version 0.13 or higher is required as it provides support for installing Terraform providers directly from the Terraform Registry.
3. The ESXi Terraform Provider built by [https://github.com/josenk/terraform-provider-esxi](https://github.com/josenk/terraform-provider-esxi) is required, but will be installed automatically during a later step.
4. Your ESXi instance must have at least two separate networks - one that is accessible from your current machine and has internet connectivity and a HostOnly network to allow the VMs to communicate over a private network. The network that provides DHCP and internet connectivity must also be reachable from the host that is running Terraform - ensure your firewall is configured to allow this.
5. [OVFTool](https://my.vmware.com/web/vmware/details?downloadGroup=OVFTOOL420&productId=618) must be installed and in your path.
    - On MacOS, I solved this by creating a symbolic link to the ovftool included in VMWare Fusion: `sudo ln -s "/Applications/VMware Fusion.app/Contents/Library/VMware OVF Tool/ovftool" "/usr/local/bin/ovftool"`
    - On Silicon Macs, OVFtool is not included in VMWare Fusion Tech Preview, so you’ll have to use the download link below and symlink via `sudo ln -s /Applications/VMware\ OVF\ Tool/ovftool /usr/local/bin/ovftool`
    - Downloads for OVFTool are also here: [https://code.vmware.com/web/tool/4.4.0/ovf](https://code.vmware.com/web/tool/4.4.0/ovf)
6. On your ESXi instance, you must:

    1. Enable SSH
    2. Enable the “Guest IP Hack”
    3. Open VNC ports on the firewall (not reqired for ESXi 7.0+)

    - Instructions for those steps are here: [https://nickcharlton.net/posts/using-packer-esxi-6.html](https://nickcharlton.net/posts/using-packer-esxi-6.html)
    - Alternatively, you can install the VIB file from [https://github.com/sukster/ESXi-Packer-VNC](https://github.com/sukster/ESXi-Packer-VNC) which will automatically open the VNC ports on the ESXi firewall.
7. [Install Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) and pywinrm via `pip3 install ansible pywinrm --user` or by creating and using a virtual environment.
8. [Packer](https://developer.hashicorp.com/packer/install) v1.7.0+ must be installed and in your PATH, packer plugin for ESXi is also required: `packer plugins install github.com/hashicorp/vmware`.
9. [sshpass](https://linux.die.net/man/1/sshpass) must be installed to allow Ansible to use password login. On MacOS, install via `brew install hudochenkov/sshpass/sshpass` as `brew install sshpass` does not allow it to be installed.

## Steps

1. **(5 Minutes)** Edit the variables in `DetectionLab/ESXi/Packer/variables.json` to match your ESXi configuration. The `esxi_network_with_dhcp_and_internet` variable refers to any ESXi network that will be able to provide DHCP and internet access to the VM while it’s being built in Packer. This is usually **VM Network**. [![variablesjson](../img/ESXi%20%20DetectionLab/variablesjson.png)](https://clo.ng/img/2020/11/variablesjson.png)

#### Special Configuration for ESXi 6.x

If you’re using ESXi 6.x (as opposed to 7.x), remove the following two directives from `builders` array:

- “vnc\_over\_websocket”: true,
- “insecure\_connection”: true,

```yml
{
  "builders": [
    {
      "vnc_over_websocket": true,    <---- Remove
      "insecure_connection": true,   <---- Remove
      "vnc_disable_password": true,
      "keep_registered": true,
...
```

to each of the following files:

- DetectionLab/ESXi/Packer/windows\_10\_esxi.json
- DetectionLab/ESXi/Packer/windows\_2016\_esxi.json
- DetectionLab/ESXi/Packer/ubuntu2004\_esxi.json

The remaining steps on this page apply to both both ESXi 6.x and 7.x:

2. **(45 Minutes: Build vm templates use Packer)** From the `DetectionLab/ESXi/Packer` directory, run:

- `PACKER_CACHE_DIR=../../Packer/packer_cache packer build -var-file variables.json windows_10_esxi.json`
- `PACKER_CACHE_DIR=../../Packer/packer_cache packer build -var-file variables.json windows_2016_esxi.json`
- `PACKER_CACHE_DIR=../../Packer/packer_cache packer build -var-file variables.json ubuntu2004_esxi.json`

These commands can be run in parallel from three separate terminal sessions.

[![Packer](../img/ESXi%20%20DetectionLab/esxi_packer.png)](https://github.com/clong/DetectionLab/blob/master/img/esxi_packer.png?raw=true)

3. **(1 Minute)** Once the Packer builds finish, verify that you now see Windows10, WindowsServer2016, and Ubuntu2004 in your ESXi console [![Ansible](../img/ESXi%20%20DetectionLab/esxi_console.png)](https://github.com/clong/DetectionLab/blob/master/img/esxi_console.png?raw=true)

4. **(5 Minutes)** In `DetectionLab/ESXi`, [Create a terraform.tfvars file](https://www.terraform.io/docs/configuration/variables.html#variable-definitions-tfvars-files) (RECOMMENDED) to override the default variables listed in variables.tf. [![variablestfvars](../img/ESXi%20%20DetectionLab/variablestfvars.png)](https://clo.ng/img/2020/11/variablestfvars.png)

5. **(25 Minutes: Power on lab machines)** From `DetectionLab/ESXi`, run `terraform init`. The [ESXi Terraform provider](https://github.com/josenk/terraform-provider-esxi) should install automatically during this step:

```bash
$ terraform init

Initializing the backend...

Initializing provider plugins...
- Finding josenk/esxi versions matching "1.8.0"...
- Installing josenk/esxi v1.8.0...
- Installed josenk/esxi v1.8.0 (self-signed, key ID A3C2BB2C490C3920)

Partner and community providers are signed by their developers.
If you'd like to know more about provider signing, you can read about it here:
https://www.terraform.io/docs/plugins/signing.html

Terraform has been successfully initialized!
```

6. Running terraform apply should then prompt us to create the logger, dc, wef, and win10 instances:

```bash
$ terraform apply
<snip>
Plan: 4 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.
```

If an ESXi server is managed by a vCenter, terraform will fail with: `Access to resource settings on the host is restricted to the server that is managing it: xx.xx.xx.` To allow terraform to work, a user can SSH into the ESXi server and run: `/etc/init.d/vpxa stop; /etc/init.d/hostd restart` This will disconnect the host from the vCenter. Once the terrafrom is complete, run `/etc/init.d/vpxa start` to reconnect with the vCenter server.

Once finished, you should see something like the following: [![tfdone](../img/ESXi%20%20DetectionLab/terraform-done.png)](https://clo.ng/img/2020/11/terraform-done.png)

7. Once Terraform has finished bringing the hosts online, change your directory to `DetectionLab/ESXi/Ansible`
8. **(1 Minute)** Edit `DetectionLab/ESXi/Ansible/inventory.yml` and replace the IP Addresses with the respective IP Addresses of your ESXi VMs. At times, the Terraform output is unable to derive the IP address of hosts, so you may have to log into the ESXi console to find that information and then enter the IP addresses into `inventory.yml` [![console](../img/ESXi%20%20DetectionLab/ip.png)](https://clo.ng/img/2020/11/ip.png) [![inventory](../img/ESXi%20%20DetectionLab/inventory.png)](https://clo.ng/img/2020/11/inventory.png)
9. **(3 Minute)** Before running any Ansible playbooks, I highly recommend taking snapshots of all your VMs! If anything goes wrong with provisioning, you can simply restore the snapshot and easily debug the issue.
10. **(30 Minutes: Setup lab machines)** Run `ansible-galaxy install -r requirements.yml && ansible-playbook -v detectionlab.yml`. This will provision the hosts one by one using Ansible. If you’d like to provision each host individually in parallel, you can use ansible-playbook -v detectionlab.yml –tags “\[logger|dc|wef|win10\]” and run each in a separate terminal tab.

    If running Ansible causes a `fork()` related error message, set the following environment variable before running Ansible: `export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES`. More about this [here.](https://github.com/clong/DetectionLab/issues/543).

11. If all goes well, you should see the following and your lab is complete! [![Ansible](../img/ESXi%20%20DetectionLab/esxi_ansible.png)](https://github.com/clong/DetectionLab/blob/master/img/esxi_ansible.png?raw=true)

If you run into any issues along the way, please open an issue on Github and I’ll do my best to find a solution.

## Configuring Windows 10 with WSL as a Provisioning Host

Note: Run the following commands as a root user or with sudo

1. In Windows 10 install WSL (version 1 or 2)
2. Install Ubuntu 18.04 app from the Microsoft Store
3. Update repositories and upgrade the distro: apt update && upgrade
4. Ensure you will install the most recent Ansible version: apt-add-repository –yes –update ppa:ansible/ansible
5. Install the following packages: apt install python python-pip ansible unzip sshpass libffi-dev libssl-dev
6. Install PyWinRM using: pip install pywinrm
7. Install Terraform and Packer by downloading the 64-bit Linux binaries and moving them to /usr/local/bin
8. Install VMWare OVF tool by downloading 64-bit Linux binary from my.vmware.com and running it with “–eulas-agreed” option
9. Download the Linux binary for the Terraform ESXi Provider from [https://github.com/josenk/terraform-provider-esxi/releases](https://github.com/josenk/terraform-provider-esxi/releases) and move it to /usr/local/bin
10. From “DetectionLab/ESXi/ansible” directory, run: “ansible –version” and ensure that the config file used is “DetectionLab/ESXi/ansible/ansible.cfg”. If not, implement the Ansible “world-writtable directory” fix by going to running: “chmod o-w .” from “DetectionLab/ESXi/ansible” directory.

## Future work required

- It probably makes sense to abstract all of the logic in `logger_bootstrap.sh` into individual Ansible tasks
- There’s a lot of areas to make reliability improvements
- I’m guessing there’s a way to parallelize some of this execution: [https://medium.com/developer-space/parallel-playbook-execution-in-ansible-30799ccda4e0](https://medium.com/developer-space/parallel-playbook-execution-in-ansible-30799ccda4e0)

## Debugging / Troubleshooting

- If an Ansible playbook fails, you can pick up where it left off with `ansible-playbook -v detectionlab.yml --tags="<hostname>" --start-at-task="taskname"`

## Credits

As usual, this work is based off the heavy lifting that others have done. My primary sources for this work were:

- [Josenk’s Terraform-ESXI-Provider](https://github.com/josenk/terraform-provider-esxi) - Without this, there would be no way to deploy DL to ESXi without paying for VSphere. Send him/her some love 💌
- [Automate Windows VM Creation and Configuration in vSphere Using Packer, Terraform and Ansible - Dmitry Teslya](https://dteslya.engineer/automation/2019-02-19-configuring_vms_with_ansible/#setting-up-ansible)
- [Building Virtual Machines with Packer on ESXi 6 - Nick Charlton](https://nickcharlton.net/posts/using-packer-esxi-6.html)
- [The DetectionLab work that juju4 has been doing on Azure and Ansible](https://github.com/juju4/DetectionLab/tree/devel-azureansible/Ansible)
- [lofi hip hop radio - beats to relax/study to](https://www.youtube.com/watch?v=5qap5aO4i9A) 🔉

Thank you to all of the sponsors who made this possible!

![Overview](../img/ESXi%20%20DetectionLab/esxi_overview.jpeg)
![Packer](../img/ESXi%20%20DetectionLab/packer.png)
![Vagrant](../img/ESXi%20%20DetectionLab/vagrant.avif)
