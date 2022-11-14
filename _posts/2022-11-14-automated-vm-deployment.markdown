---
layout: single
title:  "Automated VM deployment using Ansible and LXD"
date:   "2022-11-14"
categories: ansible
---
# Summary
1. I used Ansible to configure some spare PCs
2. These PCs run VMs and containers using LXD
3. All devices (bare-metal and virtualized) use TailScale to provide me with a flat (and secure) network
4. For web-accessible resources, I use Cloudflare Tunnels to expose self-hosted applications
5. All the above occur with zero configuration on my home router. No port forwarding, Dynamic DNS updates, or reserved DHCP MAC records

This blog post is a brief write-up on my experiences using Ansible to provision machines. I only go into a bit of detail about how I achieved this; however, I share the repository I created while working on this project. I'd also like to make an informal video where I peek through the code and explain Ansible's workings more specifically. Stay tuned!

Getting back on track, I use this blog post to describe some general background information, my approach to managing my personal infrastructure, and some general lessons that I think that others can also learn from.

# Background
When I worked in my College’s IT department, my employer let me take home some PCs that were thrown out. I recently learned Ansible at my last co-op at Descartes Systems Group. I configured these PCs with Ansible, and this write-up explains my experience doing so, including some lessons that apply to infrastructure management and some more general lessons. 

I have posted my Ansible playbooks and roles that I developed: See [here](https://github.com/ehharvey/ansible). My code is readable but needs improvement. I want to migrate my local roles to be in Ansible collections. I also plan to use Terraform instead of implementing my own VM creation role. I share my code now since it is usable, but I plan to improve it.


I had a few main goals when starting this project:
1. I wanted a set of VM hosts
2. I wanted to access my VM hosts from anywhere using the Internet
3. I wanted each VM to be identical but not clustered. Each VM host will contain a unique set of VMs (the VMs themselves could cluster if needed)
4. I wanted to automate as much as feasible
5. I have a domain (emilharvey.ca) where I want to expose select services on


# Approach
I have several old PCs. These machines ran Haswell processors. Each machine had 4-16 GB of RAM and 100-500 GB of solid-state storage. I then decided what I wanted to use these machines for.

I wanted to run Linux on each machine and use the machines as independent VM hosts. I settled upon Ubuntu Server with LXD. Ubuntu Server is popular enough, which makes finding support easy. LXD is software that runs containers and virtual machines.

All you need are virtualization hosts. You can launch whichever service you need inside a virtual machine. LXD also supports containers too. Many virtualization software, including LXD, supports clustering, but I did not find clustering to be necessary.

Networking is an important consideration. I decided to avoid complications and use Tailscale. Tailscale is a VPN. It exposes hosts and only the hosts that Tailscale is installed on. Each host receives a static IP that other Tailscale hosts can reach. Tailscale lets me have my virtual machine hosts and their virtual machines to be on the same flat virtual network. Beyond that, Tailscale acts as a firewall by letting me configure ACLs (Access Control Lists) via their web app and REST API.

{% highlight json %}
{
  // Access control lists.
  "acls": [
    // Match absolutely everything.
    // Comment this section out if you want to define specific restrictions.
    {"action": "accept", "src": ["tag:admin"], "dst": ["*:*"]},
    {"action": "accept", "src": ["tag:sqiii"], "dst": ["tag:sqiii:*"]},
  ],
  "tagOwners": {
    "tag:admin": ["xxx@gmail.com"],
    "tag:sqiii": ["xxx@gmail.com"],
  },
}
{% endhighlight %}
*Tailscale's ACL: simple JSON that describes which machines can connect to each other*

A related concern is allowing public access to services. For example, I host a SonarQube instance used by an assignment. I provided public access via this URL: https://sqiii-sonarqube.emilharvey.ca. I did so using Cloudflare Tunnels. Tunnels is an application that Cloudflare provides to let you expose services to the internet via a tunnel. I simply run the application with the appropriate configuration and update my Cloudflare DNS to point to a specially-defined URL. I can even protect access to the service via Cloudflare Access. I can use services like GitHub as identity providers. I can whitelist users by their emails, which gets proven by GitHub. The Cloudflare platform handles the rest. It shows a login screen, I authenticate via my GitHub account, and Cloudflare presents me with my self-hosted service.

Both Tailscale and Cloudflare’s services operate with zero additional network configuration. I don’t use port forwarding. I also have a dynamic Internet IP. Neither factors are a problem. I can access all of my machines via Tailscale, and the internet can access my web services via Cloudflare.

I provisioned these machines using Ansible. Ansible is a configuration management framework. It operates as a Python application that connects to other machines (usually over SSH). Ansible then runs commands and copies Python code for the remote host to execute.

One final consideration is secret management. Tailscale and Cloudflare provide API keys which I must keep secret. For simplicity, I used `passwordstore`. It lets me encrypt secrets via `GPG` (convenient since I have  Yubikey). Ansible provides a builtin look-up plugin that lets met retrieve these secrets as playbooks run.

{% highlight yaml %}
{% raw %}
tailscale_authkey: "{{ lookup('ansible.builtin.passwordstore', 'tailscale/authkey' ) }}"
lxd_trust_password: "{{ lookup('ansible.builtin.passwordstore', 'lxd/trust_password' }}"
{% endraw %}
{% endhighlight %}
*Some secrets passed to be Ansible variables. Look, no secrets!*

#  Installing Ubuntu Server
I manually installed Ubuntu Server on each machine. There are methods to automate this process, like using, but I avoided these after spending some time using Canonical’s MAAS. The approach taken by MAAS prefers it to act as a DHCP server, among other things. It is not that MAAS wouldn’t work, it just was developed with a different user than me. I found it to require , and I worry about what vulnerabilities would arise from using a required provisioning stack for MAAS, like Intel AMT and PXE. Since the virtualization hosts only run virtual machines, I didn’t think I would be reinstalling often.

During installation, I assigned static IPs to each machine. My home router/WAP device also provides DHCP, but I set static IPs so that I can easily access them later. On my home router, I set the DHCP range to be from `xxx.xxx.xxx.51` to `xxx.xxx.xxx.199` (my subnet size is `24`). My router occupied `.1`. This left me with assigning new servers from `xxx.xxx.xxx.2` to `xxx.xxx.xxx.50` and `xxx.xxx.xxx.200` to `xxx.xxx.xxx.254`. After these servers get provisioned, I’ll set them to have DHCP-provided IPs. Since I use Tailscale, I can access them through their Tailscale DNS name or Tailscale IP.

I later found that my virtualization hosts retained their static IP. Even as these devices rebooted, the router handed them their old static IPs even though they were outside the normal DHCP range. You should also be able to renew your DHCP leases to force the virtualization hosts to receive proper DHCP IPs. However, I still need to do so.

When installing Ubuntu Server, I enabled the SSH server and instructed my machine to retrieve my SSH public keys from GitHub. I created an SSH key there a while ago, and GitHub is perfectly usable as a personal key server. We can use Ansible if we need to rotate the keys.

# Provisioning Using Ansible
Now that each of my servers runs Ubuntu Server, I can further provision them using Ansible. Ansible has several concepts that work together to configure a system. We can install packages, configure services, and even run arbitrary commands. You can think of it as a framework. I defer a proper introduction to its source: see https://docs.ansible.com/ansible/latest/user_guide/intro.html. 

I first used Ansible to install Tailscale. Tailscale provides a convenient install script. I wrote tasks for Ansible to install and run it. Tailscale can be authenticated with an authentication key which you can retrieve from Tailscale’s web UI. After installing Tailscale, further Ansible tasks instructed Ansible to use Tailscale’s IPs moving forward. Afterwards, I configure each virtualization host to use a DHCP-provided IP. Since I have the Tailscale IP, I do not care what the physical IPs become.

{% highlight yaml %}
---
- name: Configure tailscale
  hosts: unprovisioned_baremetals
  roles:
    - tailscale
  vars:
    static_ips:
      baremetal0: # Static IP here
      baremetal1: # Static IP here
    ansible_host: "{{ static_ips[inventory_hostname] }}"
{% endhighlight %}
*I use an Ansible role to install Tailscale on all my virtualization hosts (I call them baremetals). Since my devices have static IPs initially, I let Ansible know which IP to use through the static_ips dictionary*

Next, I set the hostname, installed a firewall (UFW), and set the timezone, all using Ansible. 
One server had 4 disks attached as well as a main OS disk. I installed ZFS on this and created a RAID array using these 4 devices. I next Ansible to install LXD. That’s it: now I have one or more virtualization hosts! Right now, I have 2, but I can rerun the Ansible tasks on new machines as necessary.

With Tailscale, networking becomes simple. Physical ethernet interfaces can have a simple `deny all incoming` rule. Tailscale creates a new, virtual interface called `tailscale0`. I set the firewall to permit incoming `SSH` (port 22) only. I needed no other configuration: even if I want to host a website, I can use Cloudflare Tunnels to expose the website via a tunnel.

## Creating a Virtual Machine
Now I am ready to launch virtual machines. I use Ansible to create virtual machines. LXD defaults to providing bridged networking to each guest. Each guest can receive the Internet via NAT. For those unfamiliar, this is similar to how most home networks work. You have a private network; a typical network being a `192.168.x.0/24`. Your router provides NAT to give each private network device the internet.


{% highlight yaml %}
---
- name: Create MySQL VM
  hosts: baremetal1
  tasks:
  - name: Create MySQL VM
    ansible.builtin.include_role: 
      name: lxdvm
    vars:
      lxd:
        name: sqiii-mysql
        config:
          limits.memory: 1GB
        group: "sqiii"
{% endhighlight %}


*Ansible Playbook to create a virtual machine for a SonarQube Instance*

When we remember networking basics, we see that this approach allows each home network device (and LXD virtual guest) “one-way” access to the internet. You can initiate connections to Google directly, but Google cannot do the same to you. Tailscale and Cloudflare Tunnels mitigate this by making hosts continually initiate connections (to Cloudflare or to each other).

I use Ansible to create new virtual machines. LXD lets me run arbitrary commands on virtualization guests, letting me install Tailscale immediately after the virtualization guest is created. I can then provision the virtualization guest just like my virtualization host, using the same Ansible tasks to set the firewall, etc.

The way I used Ansible somewhat overlaps with another system called Terraform. Terraform specializes on virtual machine 

I can use Ansible to launch a new virtualization that I install SonarQube to. I can access it via Tailscale, but others can not. To make SonarQube publicly accessible, I then turned to Cloudflare Access and Tunnels. I created a tunnel which proxied my self-hosted instance to Cloudflare. I then created access controls to only let me access it. SonarQube initially starts with weak default credentials. I completed the setup (and updated the credentials). Afterwards, I manually deleted the Cloudflare Access rules. Now, SonarQube is accessible all.

# Lessons Learned and Next Steps
Here, I describe some general lessons that I learned. In brief, my experience here reminded me to not try to reimplement past work. I also see the value of being able to fail fast and repeat attempts quickly.

The main lesson I learned here was to try applying what exists instead of trying to reinvent your own implementation. I struggled with networking for a while before discovering Tailscale. The same applies to Cloudflare. I am exploring Terraform now since I feel that my approach to VM management is not easily scalable. I am also hoping to explore other VM-related improvements, such as creating pre-configured images.

A general software engineering lesson that applies here is that you want easy testability. In general software development, you do so with {unit, integration, system, etc.} testing. In system administration, you can do so via personal virtual machines (on your laptop, etc.) as you start out. Aiming to have your physical hosts simply be virtual machine hosts is also great since you can easily recreate virtual machines. The ease that Tailscale and Cloudflare provide means minimal interdependent configurations to manage, such as firewalls and routing tables, etc.

Finally, as I mentioned earlier, my Ansible code lives in a haphazard state on a single repository. In my project class, I've successfully implemented automated tests and other CI/CD components in my [current project class](https://github.com/CSCN73030-projectv-group9/Attendance). I've yet to apply the same organization approach with my Ansible work, but after seeing Ansible's Collection, I see the benefit to doing so.