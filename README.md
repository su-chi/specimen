# Objective
Write a script to provision a new hypothetical application server on a cloud service provider.

1. Must be able to run docker as non-root.
2. Must be able to accept web traffic on port 80/443.
3. Must be locked down, key authentication only, no root access.
4. Prefer if we used Ubuntu images.

This script can be in any language, but be able to speak to each point. For this example, a working demo would be great. You can use any docker image you’d like to run a basic web server (hello world)

# How to use
To use these playbooks, you need:
- Install `ansible` and have `ansible-playbook` in your `$PATH`
- Have a [DigitalOcean](https://www.digitalocean.com/) account and an API token with write access

## Build & destroy primary instance
(NOTE: this could easily be extended create a pool of "primary" instances)

Build:
```
$ DO_API_TOKEN=YOURDOAPITOKENHERE ansible-playbook build_instance.yaml
```

Destroy:
```
$ DO_API_TOKEN=YOURDOAPITOKENHERE ansible-playbook destroy_instance.yaml
```

## Build & destroy a PR instance
Build:
```
$ DO_API_TOKEN=YOURDOAPITOKENHERE PR_NUMBER=1440 ansible-playbook build_instance.yaml
```

Destroy:
```
$ DO_API_TOKEN=YOURDOAPITOKENHERE PR_NUMBER=1440 ansible-playbook destroy_instance.yaml
```

# Solution
This repository contains two ansible playbooks - `build_instance.yaml` and `destroy_instance.yaml` - for building, configuring and destroying a DigitalOcean droplet.

## `build_instance.yaml`
This playbook is divided into 3 separate plays:
- Play 1 - runs on `localhost`
  - Build DigitalOcean droplet w/ my public SSH key setup on the `root` user
  - Once the droplet is built, capture's the public IP address
  - Creates an A record pointing to the new droplet's IP address
- Play 2 - runs as the `root` user on the new droplet
  - This play will ONLY run if the droplet was just built or changed
  - Executes tasks in the `boostrap` role
    - Creates `config` user with `NOPASSWD` sudo access
    - Copies my public SSH key from the root user to the `config` user
    - Disables SSH root login and password authentication
    - Restarts sshd
- Play 3 - runs as the `config` user on the new droplet
  - This play will always run. It handles the main configuration of the server
  - Executes tasks in the `docker` role
    - Installs dependencies for docker-ce
    - Sets up the docker-ce repository
    - Installs docker-ce
    - Creates `appuser` user that is in the `docker` group
  - Executes tasks in the `ufw` role
    - Allows traffic on ports 22, 80, and 443
    - Enables ufw and sets the incoming default policy to `deny`
  - Spins up a vanilla `nginx:latest` container with port mappings for port 80 & 443
  - Prints the URL to access the nginx container

## `destroy_instance.yaml`
This playbook is much more simple. It only needs to run from `localhost` just destroys the droplet and associated A record.

## Environment Variables
These playbooks can utilize the following environment variables:
- `DO_API_TOKEN`: Required. This is the DigitalOcean API token.
- `PR_NUMBER`: Optional. If `PR_NUMBER` is set, the droplet instance name and DNS will include it: `speciment-1920`

## Areas for improvement
To avoid using multiple tools in my solution, I decided to design the `build_instance.yaml` playbook to handle every part of the process - building the cloud resources (droplet), configuring it/them, creating DNS, etc... Ansible's real strength is in configuration management. It's technically capable of creating cloud resources, but an Infrastructure as Code (IaC) tool - like Terraform - is better suited for this need. I'll cover a hypothetical alternate solution later.

Supporting the `PR_NUMBER` environment variable would give this the ability to spin up & configure instances for specific PRs. Obviously there's more work that needs to go into the container image that it's running.

If this were to grow into an actual deployment tool, we would want to parameterize a lot more to make it more customizable. There are also several other configuration changes that we would probably want to make on the droplet. The nice thing about this design is, we could create new roles (similar to the `docker` and `ufw` roles) and include them in the `roles` list of the last play in `build_instance.yaml`.

# A more scalable solution
As I mentioned before, an Infrastructure as Code tool like Terraform would be better suited for spinning up the cloud resources. The thing that's nice about this current implementation is, the ansible roles that are used to configure the instance could still be used. Here's how I would design this:

- Use [Packer](https://www.packer.io/) to build an AMI (Amazon Machine Image) using existing ansible role(s)
- Push the new AMI to appropriate AWS region(s)
- Use Terraform to:
  - Create a [Launch template](https://docs.aws.amazon.com/autoscaling/ec2/userguide/launch-templates.html) that can be configured to reference the AMI built via Packer & Ansible
  - Create an [Auto Scaling Group (ASG)](https://docs.aws.amazon.com/autoscaling/ec2/userguide/auto-scaling-groups.html) that uses the Launch template from the previous step to spin up new EC2 instances.
  - Create an [Elastic Load Balancer (ELB)](https://aws.amazon.com/elasticloadbalancing/) that's configured to balance traffic between nodes in the ASG from the previous step.

This design scales well because the slow part - configuring the instances - only has to happen once, then you can reuse the AMI to build pre-configured instances on demand. If we notice that we're getting more traffic than we can handle, we just scale up the ASG and it automatically builds new nodes that will get plopped into the ELB and start serving traffic.
