# Vagrant

## Overview

Vagrant is a tool for building and managing virtual development environments. It uses a `Vagrantfile` to define Virtual Machine (VM) configurations and works with providers such as VirtualBox or VMware to create and manage VMs.

### Why use Vagrant?

Vagrant allows environments to be defined as code, making it possible to recreate consistent environments across different machines. This reduces environment-related problems between developers, such as "It works on my machine."

### Key Features

- **Reproducible Environments:** Define configurations that can be used to recreate environments consistently.
- **Multiple Virtual Machines:** Define and manage multiple VMs within a single project.
- **Provisioning:** Automatically install software and configure systems using Shell scripts or tools such as Ansible.
- **Resource Management:** Configure resources such as CPU and memory through supported providers.
- **Networking:** Configure communication between VMs and port forwarding between the host and guest.
- **SSH Access:** Easily connect to VMs through SSH using `vagrant ssh`.

## Architecture & Workflow

### Main Components

- **Vagrantfile:** A configuration file written using Ruby syntax that defines VM and environment settings.
- **Box:** A template package used as the base for creating VMs. It may contain an operating system and baseline configurations.
- **Provider:** The software or system Vagrant communicates with to create and manage VMs, such as VirtualBox or VMware.
- **Virtual Machine (VM):** A virtualized machine that serves as the working environment.
- **Provisioner:** A tool or script used to install software and configure the environment inside a VM.

### How It Works

1. The user defines the environment configuration in a `Vagrantfile`.
2. Vagrant reads the configuration and checks the required Box.
3. Vagrant communicates with the provider to create or start the VM.
4. Vagrant configures networking and resources according to the defined settings.
5. Vagrant executes provisioners if they are configured.
6. The user accesses the VM through SSH and begins working.

### Workflow Overview

`Vagrantfile` → Vagrant → Provider → Virtual Machine → Provisioning → Ready Environment

## Configuration Reference

### 1. Initialize a Project

Create an initial `Vagrantfile` using:

```bash
vagrant init bento/ubuntu-24.04
```

Basic configuration:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-24.04"
end
```

- `Vagrant.configure("2")` specifies the Vagrant configuration schema version.
- `config.vm.box` specifies the Box used to create the VM.

To specify a particular Box version, use `config.vm.box_version`. Make sure the requested version exists.

### 2. Provisioning

Provisioning automates software installation and environment configuration inside a VM.

Example using inline Shell commands:

```ruby
config.vm.provision "shell",
  name: "install-packages",
  inline: "sudo apt-get update"
```

Example using a Shell script:

```ruby
config.vm.provision "shell",
  name: "setup",
  path: "scripts/setup.sh"
```

- `name` identifies the provisioner.
- `inline` specifies Shell commands directly.
- `path` specifies the script file to execute.

Run provisioning again with:

```bash
vagrant provision
```

Alternatively, run provisioning when starting the VM:

```bash
vagrant up --provision
```

### 3. Synced Folders

Synced folders share files between the host machine and the guest VM.

```ruby
config.vm.synced_folder "./project", "/home/vagrant/project",
  create: true
```

- The first path specifies the directory on the host.
- The second path specifies the directory inside the guest.
- `create: true` allows Vagrant to create the destination directory in the guest if it does not exist, subject to the capabilities of the selected synced folder type.

Consider file permissions and filesystem performance when choosing a synchronization method.

### 4. Hostname

Set the VM hostname:

```ruby
config.vm.hostname = "web-server"
```

A hostname identifies the machine within an environment, but it does not automatically enable communication between VMs.

### 5. Networking

#### Private Network

Configure an IP address for communication through a private network:

```ruby
config.vm.network "private_network",
  ip: "192.168.56.10"
```

Choose an IP address that does not conflict with existing networks and verify the requirements of the selected provider.

#### Forwarded Port

Forward a port from the host to the guest:

```ruby
config.vm.network "forwarded_port",
  guest: 8080,
  host: 8080
```

This configuration allows a service listening on port `8080` in the guest to be accessed through port `8080` on the host, provided the service and networking are configured correctly.

For UDP traffic:

```ruby
config.vm.network "forwarded_port",
  guest: 5353,
  host: 5353,
  protocol: "udp"
```

If the host port is already in use, `auto_correct: true` allows Vagrant to attempt to select another available host port:

```ruby
config.vm.network "forwarded_port",
  guest: 8080,
  host: 8080,
  auto_correct: true
```

#### Public Network

A public network connects a VM to an external network, such as a LAN, through the provider's networking mechanism. Depending on the environment, additional network interface selection and IP configuration may be required.

### 6. Multiple Virtual Machines

Vagrant can define multiple VMs in a single `Vagrantfile`:

```ruby
Vagrant.configure("2") do |config|
  config.vm.define "web" do |web|
    web.vm.box = "bento/ubuntu-24.04"
    web.vm.hostname = "web"
  end

  config.vm.define "redis" do |redis|
    redis.vm.box = "bento/ubuntu-24.04"
    redis.vm.hostname = "redis"
  end
end
```

- `web` and `redis` are the names assigned to the VMs within Vagrant.
- The `web` and `redis` block variables are used to configure their respective VMs.
- Each VM can have its own networking, provisioning, and resource configuration.

Creating multiple VMs does not automatically enable communication between them. Appropriate networking and connectivity settings are still required.

### 7. Provider Configuration

Provider configuration controls VM properties that depend on the selected hypervisor or provider.

Example for VirtualBox:

```ruby
config.vm.provider "virtualbox" do |vb|
  vb.memory = 2048
  vb.cpus = 2
end
```

This example allocates 2048 MB of memory and two virtual CPUs. Verify that the selected options are supported by the installed provider and its version.

### 8. Environment Variables and Secrets

Ruby can read environment variables from the environment in which Vagrant runs:

```ruby
secret_token = ENV["KUBE_TOKEN"]
```

If you use a plugin such as `vagrant-env`, install and configure it according to its documentation before using its features.

Security considerations:

- Avoid hardcoding secrets in the `Vagrantfile`.
- Do not commit files containing secrets to Git.
- Validate that required environment variables are present before using them.
- Reading an environment variable on the host does not automatically transfer the secret into the guest VM.

## Common Workflow

### 1. Initialize

```bash
vagrant init bento/ubuntu-24.04
```

Creates an initial `Vagrantfile`.

### 2. Start the Environment

```bash
vagrant up
```

Creates or starts VMs according to the configuration.

### 3. Access the VM

```bash
vagrant ssh
```

Connects to a VM through SSH.

If multiple VMs are defined, specify the VM name:

```bash
vagrant ssh web
```

### 4. Check VM Status

```bash
vagrant status
```

Displays the status of VMs managed by Vagrant.

### 5. Run Provisioning

```bash
vagrant provision
```

Runs the configured provisioners against the VM.

### 6. Stop the Environment

Stop the VM:

```bash
vagrant halt
```

Suspend the VM:

```bash
vagrant suspend
```

Resume a suspended VM:

```bash
vagrant resume
```

### 7. Destroy the Environment

```bash
vagrant destroy
```

Destroys the VMs managed by Vagrant. Data stored inside those VMs may be lost, so back up any required data beforehand.

### Important Notes

- `vagrant up` does not necessarily create a new VM every time. If a VM already exists, Vagrant may start the existing machine.
- `vagrant provision` runs provisioning again, but the result depends on the scripts and provisioners configured.
- Reproducibility requires controlling the `Vagrantfile`, Box version, provisioning scripts, and relevant dependencies.
- Consult the official Vagrant and provider documentation when using version-dependent configuration options.
