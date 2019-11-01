---
title:  "Automation in the Homelab - Ansible vs Terraform"
categories: [homelab, terraform, docker]
tags: [homelab, terraform, ansible, automation]
---
Automation for simple, self-hosted solutions

![ansible-terraform](../../assets/ansible_vs_terraform.png)

[Ansible](#Ansible) - [Terraform](#Terraform)

## A "self-hosted" time bomb
When working on a self hosted solution, or just having a small server rack in your basement, you are taking a step towards having a "production" environment that must care and feed for. In many of the posts you see on https://reddit.com/r/homelab or https://reddit.com/r/selfhosted, these are one-off solutions or designs with very little in the way of repeatability. This is not to say that all solutions are singleton, manual installs, but there is a clear lack of an automation platform for these self hosted installs. When it comes down to the grit of the issue, people tend to put off the repeatability of their own production, as it never has the level of consumers that other services in the wild expect.

If you look at self hosted this way, it will be a difficult path to maintain and, for many, the reason they stop.

Let's say you spend a full week of time setting up every single install of every server and service you would ever need;
- plex
- unifi
- couchpotato
- vpn
- reverse proxy
- ad blocker
- router configuration

And for the sake of the example, these all are running (hopefully) in containers for a slightly more repeatable manner. Now you finally have all this running on a docker-host in your server rack after painfully configuring and setting up all the proper port configurations, UI logins and other superficial, but highly important details of each, and it runs beautifully for a few months with very few changes. However, a failed disk or a power outage knocks out everything you had setup, and without a clean document of what was running or how each was setup, it will be at least another week of setup and re-configuring to be back where you were.

I shouldn't have to mention the importance of backups, so most of the sensitive data should be (assumed) safe, but what about the boilerplate services? Do you have all the docker container information, or port setups properly saved somewhere? This is where the automation tools come in.

Ansible and Terraform are really aimed for the enterprise, but they have very real world applications that can be ported into even the most modest of home lab stacks. It makes the initial stand-up, and even the configuration, a trivial process. It not only brings the benefit of disaster recovery by having declarative manifests of what is needed, but it also allows for the inital standup to be tweaked without making live changes with very little insight into the changes that could occur.

![ansible](../../assets/ansible_logo.png)
## Ansible


Ansible is an automation tool, directly targeted at IT automation. When you think a typical operations center, Ansible would probably be a somewhat common occurrence for automating infrastructure and configuration.

At a high level, Ansible works off the concept of inventory, modules, tasks and playbooks to wrap it together. I will be mainly focusing on the ansible-playbooks as a way to holistically explain everything together.
An example playbook will have the following structure in most cases:
```yaml
- hosts: myhost
  tasks:
  - name: do something 1
  - name: do something 2
```
This example, while not actually doing anything, would target any inventory named `myhost` and run the following two tasks on the host. There are a bit more configuration when running on a remote host, and for this overview I won't go into too much detail, but there will be documentation in the [reference links below](#Reference)

Basing off our example of a dockerized approach to self-hosted containers, we could start off with the need to rebuild our wireless controller for our access points, in this case I use a common approach with unifi and Ubiquity.

``` yaml
- hosts: localhost
  become: no
  tasks:
  - name: Verify Docker Module for Python
    pip:
      name: docker
  - name: Create a volume
    docker_volume:
      name: unifi
  - name: Setup Unifi Container
    docker_container:
      name: unifi-controller
      image: linuxserver/unifi-controller:5.6.42-ls54
      restart: yes
      volumes:
      - "unifi:/unifi"
      ports:
      - "3478:3478/udp"
      - "10001:10001/udp"
      - "8080:8080"
      - "8081:8081"
      - "8443:8443"
      - "8843:8843"
      - "8880:8880"
      - "6789:6789"
```

In this example, I am running the following on my local system, denoted by the target `hosts: localhost`. The `become: no` can allow a command to be run as a different target user, so if this was running on my docker-box, and I wanted the command to execute under a docker user, I could then set this to yes, and set a `become_user: docker`. This will be [linked below](#Reference)
The next section is where the bulk operations happen, and these are called tasks. Each task has a human-readable `name` field which should be used to briefly explain the operation that takes place.
Next, the task outlines exactly what happens. Let us take the first task:
``` yaml
- name: Verify Docker Module for Python
  pip:
    name: docker
```
All this is doing is using the pip module (python package management) to install/verify that `docker` is installed. This equates to a user running `pip install docker` on the command line.

``` yaml
- name: Create a volume
  docker_volume:
    name: unifi
```
This task is setting up a docker volume with the name `unifi` for persistence. There is obviously a bit more that could go into backing up this volume, or mounting a target NAS to the docker volume directory, but let's assume this is just a proof of concept, and this playbook can be expanded as much as needed.

```yaml
- name: Setup Unifi Container
  docker_container:
    name: unifi-controller
    image: linuxserver/unifi-controller:5.6.42-ls54
    restart: yes
    volumes:
    - "unifi:/unifi"
    ports:
    - "3478:3478/udp"
    - "10001:10001/udp"
    - "8080:8080"
    - "8081:8081"
    - "8443:8443"
    - "8843:8843"
    - "8880:8880"
    - "6789:6789"
```
This is where the most information comes into play. The following task will pull down the `image` referenced, set a few flags on the command (e.g. restart, volumes and ports) and run it. While it seems like a lot, it is much easier to have this documented in code, rather than typing out the full docker command:
```bash
docker run -d --restart always \
  -v unifi:/unifi \
  -p 3478:3478/udp \
  -p 10001:10001/udp \
  -p 8080:8080 \
  -p 8081:8081 \
  -p 8443:8443 \
  -p 8843:8843 \
  -p 8880:8880 \
  -p 6789:6789 \
  linuxserver/unifi-controller:5.6.42-ls54
```

Now this is a simplistic example, but here is a quick output of the example run of this playbook:
``` shell
 [WARNING]: No inventory was parsed, only implicit localhost is available

 [WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'


PLAY [localhost] *****************************************************************

TASK [Gathering Facts] ***********************************************************
ok: [localhost]

TASK [Create a volume] ***********************************************************
changed: [localhost]

TASK [Install Docker Module for Python] ******************************************
ok: [localhost]

TASK [Setup Unifi Container] *****************************************************
changed: [localhost]

PLAY RECAP ***********************************************************************
localhost                  : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Ansible Summary

Ansible really shines in the Configuration and Host space where you have dozens, if not hundreds, of hosts that all are running a targeted setup. It allows for quick fan-out configuration changes on multiple hosts within your Ansible-managed inventory. It also allows for different task playbooks for different host groups, so you can send targeted commands for AMD servers vs Intel and other situations that could beneift from this targetted approach. In practice, it doesn't deal well with management of the current state of systems. It does give very robust feedback on command execution and success, but if a command was to fail a roll-out, it would be a manual process to resolve the error on the target host.

![terraform](../../assets/terraform_logo.png)
## Terraform

Terraform, like Ansible, is another Automation tool that is targeted for cloud infrasturture management. It is touted as an "orchestration" engine for infrastructure, and is also much newer than Ansible. As of writing this, the current Terraform version is only v0.12.13.

Where Terraform really shines is in maintaining state after creation. It keeps an output of the state, in `terraform.tfstate` of everything created, and the current state of each resource.

In a terraform execution, there are 2 primary pieces that can be observed
- Providers
- Resources

Providers are the main software or platform and Resources are the definitions of what is to be created. Lets take the same example of unifi in Ansible, and translate it to a terraform file:
```hcl
provider "docker" {}

variable "unifi_container" {
  description = "Unifi controller image version/tag"
  default = "linuxserver/unifi-controller:5.6.42-ls54"
}

resource "docker_volume" "unifi" {
  name = "unifi"
}

resource "docker_container" "unifi" {
  name    = "unifi-controller"
  image   = var.unifi_container
  restart = "always"
  volumes {
    volume_name    = docker_volume.unifi.name
    container_path = "/unifi"    
  }

  ports {
    internal = 3478
    external = 3478
    protocol = "udp"
  }
  ports {
    internal = 10001
    external = 10001
    protocol = "udp"
  }
  ports {
    internal = 8080
    external = 8080
  }
  ports {
    internal = 8081
    external = 8081
  }
  ports {
    internal = 8443
    external = 8443
  }
  ports {
    internal = 8843
    external = 8843
  }
  ports {
    internal = 8880
    external = 8880
  }
  ports {
    internal = 6789
    external = 6789
  }
}
```

The way terraform works, is that a provider, in this case [Docker](https://www.terraform.io/docs/providers/docker/index.html), is used to define the upfront requirement to run. This could be another provider, and Terraform has many providers supported. These will [be linked below](#Reference)
On first glance, this file is much more verbose to accomplish the same result as Ansible, and you would be right in that observation. Terraform uses its own language called HCL or Hashicorp configuration language. It uses a format that is similar to json in a way, but also tailored specifically for Terraform. In Terraform v0.12, it also can directly reference variables without the original "${}" syntax that was needed in previous versions.

In the example above, we are expecting the same end state:
```HCL
provider "docker" {}

variable "unifi_container" {
  description = "Unifi controller image version/tag"
  default = "linuxserver/unifi-controller:5.6.42-ls54"
}
```
This code block adds the boilerplate provider definition, as well as a variable definition for the unifi_container image name. No resources are created in this block.

```HCL
resource "docker_volume" "unifi" {
  name = "unifi"
}
```
In this resource, we are declaring a docker volume. This is, again, synonymous with a user executing `docker volume create unifi`. The end state of this will be a docker volume named `unifi`

```HCL
resource "docker_container" "unifi" {
  name    = "unifi-controller"
  image   = var.unifi_container
  restart = "always"
  volumes {
    volume_name    = docker_volume.unifi.name
    container_path = "/unifi"    
  }

  ports {
    internal = 3478
    external = 3478
    protocol = "udp"
  }
  ports {
    internal = 10001
    external = 10001
    protocol = "udp"
  }
  ports {
    internal = 8080
    external = 8080
  }
  ports {
    internal = 8081
    external = 8081
  }
  ports {
    internal = 8443
    external = 8443
  }
  ports {
    internal = 8843
    external = 8843
  }
  ports {
    internal = 8880
    external = 8880
  }
  ports {
    internal = 6789
    external = 6789
  }
}
```
Once again, the bulk of this operation is in the docker container section. This uses a resource of `docker_container` to reference the variable container name, and the output of the volume. It also sets up the same ports using port blocks, which can be repeated for each exposed port, and also sets the container to `restart = always`. If this was to run on our docker-box we would get the following output:
```shell
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # docker_container.unifi will be created
  + resource "docker_container" "unifi" {
      + attach           = false
      + bridge           = (known after apply)
      + container_logs   = (known after apply)
      + exit_code        = (known after apply)
      + gateway          = (known after apply)
      + id               = (known after apply)
      + image            = "linuxserver/unifi-controller:5.6.42-ls54"
      + ip_address       = (known after apply)
      + ip_prefix_length = (known after apply)
      + log_driver       = "json-file"
      + logs             = false
      + must_run         = true
      + name             = "unifi-controller"
      + network_data     = (known after apply)
      + restart          = "always"
      + rm               = false
      + start            = true

      + ports {
          + external = 3478
          + internal = 3478
          + ip       = "0.0.0.0"
          + protocol = "udp"
        }
      + ports {
          + external = 10001
          + internal = 10001
          + ip       = "0.0.0.0"
          + protocol = "udp"
        }
      + ports {
          + external = 8080
          + internal = 8080
          + ip       = "0.0.0.0"
          + protocol = "tcp"
        }
      + ports {
          + external = 8081
          + internal = 8081
          + ip       = "0.0.0.0"
          + protocol = "tcp"
        }
      + ports {
          + external = 8443
          + internal = 8443
          + ip       = "0.0.0.0"
          + protocol = "tcp"
        }
      + ports {
          + external = 8843
          + internal = 8843
          + ip       = "0.0.0.0"
          + protocol = "tcp"
        }
      + ports {
          + external = 8880
          + internal = 8880
          + ip       = "0.0.0.0"
          + protocol = "tcp"
        }
      + ports {
          + external = 6789
          + internal = 6789
          + ip       = "0.0.0.0"
          + protocol = "tcp"
        }

      + volumes {
          + container_path = "/unifi"
          + volume_name    = "unifi"
        }
    }

  # docker_volume.unifi will be created
  + resource "docker_volume" "unifi" {
      + driver     = (known after apply)
      + id         = (known after apply)
      + mountpoint = (known after apply)
      + name       = "unifi"
    }

Plan: 2 to add, 0 to change, 0 to destroy.

Do you want to perform these actions in workspace "home"?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

docker_volume.unifi: Creating...
docker_volume.unifi: Creation complete after 0s [id=unifi]
docker_container.unifi: Creating...
docker_container.unifi: Creation complete after 1s [id=5b963dc6619300789428166c1dd14e8147246c4743a1f3f68ed00392d8a56f6f]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

### Terraform Summary
Terraform is a great tool when dealing with large infrastructure that has many interdependent parts. It can easily reference direct output of one resource as input to another and can even verify the state on additional apply. Terraform is also very state-dependent, and can import other pieces into it's "state" to avoid conflicting results.

## Summary
When it comes to a self hosted solution, or even a cloud solution, consistency is key. If you cannot have a consistent end state for your project, you will spend more time on maintenance and setup, than actually being able to enjoy what you have built. I personally have chosen Terraform for my own management of servers and services for the state management alone, but both tools have their benefits of use.
The primary takeaway should be that no matter the setup, you should have some way to roll out changes in a mature fashion.


### Reference
#### Ansible Reference
[Ansible - Getting Started](https://docs.ansible.com/ansible/latest/network/getting_started/basic_concepts.html) - Link to getting started guide with Ansible, and some basic concepts to understand the syntax

[Ansible - become documentaion](https://docs.ansible.com/ansible/latest/user_guide/become.html#become-connection-variables) - Documentation of Ansible privilege escalation
#### Terraform Reference
[Terraform - Getting Started](https://www.terraform.io/intro/index.html) - Overview on terraform and further links to providers and concepts.
[Terraform - Providers](https://www.terraform.io/docs/providers/index.html) - List of all Terraform providers
[Hashicorp - HCL](https://github.com/hashicorp/hcl) - HCL git repository
