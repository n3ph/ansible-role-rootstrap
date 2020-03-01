# Ansible Roostrap Role

## Purpose

This role implements our [priviledge escalation workflow](#workflow).

## Workflow

Our workflow for SSH access and priviledge escalation includes the following
deployment cases:
1. _(local)_ Vagrant VM
2. _(remote)_ Packer VM Build
3. _(remote)_ Direct Deployment

The following assumptions are made for simplicity:

1. Leave `NOPASSWD sudo` as the default for priviledge escalation.
2. Always use SSH user `vagrant` for Vagrant deployment.
3. Always use SSH user `ansible` for direct deployment.

### Local Vagrant VM

#### What is it?

Vagrant is used to provision a local VM (or multiple local VMs) for test
purposes.

#### How to identify this deployment type in Ansible?

Vagrant does not provide any extra variables to Ansible. The only simple
assumption we can make is that `vagrant` user is used for provisioning.

#### How is initial SSH access managed?

Vagrant automates this on its own.

#### How is priviledge escalation managed?

Vagrant assumes having NOPASSWD sudo access into the VM.

#### What action(s) do we need to take?

None. Vagrant will access the VM as `vagrant` user, Ansible will try using
`sudo` for priviledge escalation, as per defaults.

### Packer Build

####  What is it?

Packer is used to build a VM image in the cloud environment. Packer spawns a
temporary VM from an initial image, runs an Ansible provisioner, stops the VM
and saves its image.  The resulting VM image can later be re-used to spawn
multiple VM instances, e.g., in an auto-scaling group.

####  How to identify this deployment type in Ansible?

Packer provides an extra variable `packer_builder_type` to Ansible.

####  How is initial SSH access managed?

The SSH user name is specific to the source image and should be provided in the
Packer JSON file.

The public SSH key of the local user is supplied to the cloud provider and
registered in the spawned VM.

####  How is priviledge escalation managed?

`NOPASSWD sudo` is expected by Packer and supplied by cloud instance
provisioners.

####  What action(s) do we need to take?

Once we spawn a VM from a Packer-build image, we need to access it as user
`ansible` with our SSH keys and use `su` with password for priviledge
escalation. We can not use a host-specific password here, because the image is
generic. Therefore, we need to:

1. Deploy Ansible user with our SSH keys
2. Set root password to a "default" one.

After this VM image will be spawned as a VM instance, we may change the root
password to a host-specific one during the next Ansible deployment.

### Direct Deployment

####  What is it?

Any deployment on a running server.

####  How to identify this deployment type in Ansible?

Check that it's neither Vagrant nor Packer deployment type.

####  How is initial SSH access managed?

* When a new physical server is bootstrapped, we will use some default SSH
  credentials and priviledge escalation to access it
* When a new VM is spawned from a Packer-build image, it has everything set up
  already

####  How is priviledge escalation managed?

Su + root password from the password store. On a newly spawned VM, the password
will be the default one, as set up by Packer. Rootstrap role will update the
default password with a host-specific one.

####  What action(s) do we need to take?

1. Check the default priviledge escalation.
2. Otherwise, check priviledge escalation with host-specific root password.
3. Otherwise, check priviledge escalation with default root password.
4. If that fails, report an exception.

5. When either of these conditions is met:

    - Packer build
    - SSH user is not `ansible` (e.g., we are bootstrapping a new server)
    - tag `-t rootstrap` is provided explicitly

   deploy `ansible` user with our SSH keys and change the root password to the
   host-specific one.


## Usage

1. Install [Ansible plugins](https://some.link.tld/to/ansible/plugins).

2. Customize your per-project or global `ansible.cfg`:
```
# default remote user to 'ansible' (can be overridden via `--user`)
remote_user = ansible
# disable fact gathering by default, as it is triggered in the rootstrap role
gathering = explicit
```

3. Specify paths to the custom and the default root passwords

4. Prepend rootstrap role to the role list of each play with `tags: always`.

## Future work

In future we could replace this role with a `vars_plugin` similar to [this
one](https://gist.github.com/mfriedenhagen/e488235d732b7becda81). In that case
we would not need a role anymore, but just to add the path to this plugin to
`ansible.cfg`.
