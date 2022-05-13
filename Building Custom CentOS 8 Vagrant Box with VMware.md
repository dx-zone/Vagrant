### Building Custom CentOS 8 Vagrant Box Using VMware as a Virtualization Provider



**Building a box**

First, we are going to create a virtual machine with VMware Workstation/Fusion, then setup the VM to meet Vagrant requirements, then package the VM into a Vagrant box.



**Prerequisite**

* VMware Workstation or VMware Fusion
* Vagrant
* Vagrant `vagrant-vmware-desktop ` Plug-in
* CentOS 8 ISO



**Prepare the VM**

* Create the virtual machine in VMware Workstation/Fusion

* Install the desired Linux OS

* Enable SSH access

* Add  `vagrant` as user and set `vagrant` as password

  ```bash
  useradd vagrant
  echo "vagrant" | passwd vagrant --stdin
  # type 'vagrant' as password
  ```

* Set `vagrant` user as super user without password prompt in the sudoer directory

  ```bash
  cat << EOF > /etc/sudoers.d/vagrant
  # add vagrant user
  vagrant ALL=(ALL) NOPASSWD:ALL
  EOF
  ```

* Configure the hostname

  Replace `centos8-vagrant-box` with whatever hostname you prefer

  ```bash
  hostnamectl set-hostname centos8-vagrant-box
  cat << EOF >> /etc/hosts # Append the hostname in the /etc/hosts file
  127.0.0.1   centos8-vagrant-box
  EOF
  ```

* Generate SSH keys for `vagrant` user and copy the public key into the same VM

  ```bash
  su - vagrant
  ssh-keygen -t rsa -b 4096 -f /home/vagrant/.ssh/id_rsa -N ''
  ssh-copy-id -i /home/vagrant/.ssh/id_rsa.pub vagrant@localhost
  sudo -s
  ```

* Install VMware Tools on the VM (if VMware Workstation/Fusion didn't installed it automatically). To check the VMware tools status execute the following command:

  ```bash
  systemctl status vmtoolsd.service
  # The status should be "Active: active (running)..."
  ```



**Package the VM into a Vagrant box format**

In the system were the VMware Workstation/Fusion is installed, powering the VM, were Vagrant is also installed, proceed with the following steps to pack the VM into a Vagrant box format: 

* Since we generated the SSH keys inside the VM itself, we need to have copies of keys outside the VM as we will use them later when launching of Vagrant boxes.

  ```bash
  mkdir ./keys
  cd ./keys
  # Copy the SSH keys out of the VM into our secured key folder
  scp vagrant@<vm_ip_address>:/home/vagrant/.ssh/id_rsa* ./
  exit
  ```

* It is safe to power of the VM at this stage to start packing it up

  ```bash
  # SSH into the VM and execute the poweroff command to turn the VM off
  ssh root@<vm_ip_address> 'poweroff'
  ```

* Optimize the VM disk (Box Size)

  ```bash
  vmware-vdiskmanager -d /path/to/main.vmdk
  vmware-vdiskmanager -k /path/to/main.vmdk
  ```

* Package the VM
  Replace `~/vmware/centos8/` to point to the current location were the VM files in your systems are stored. By default the VMware stores the VMs in the `vmware` directory inside your home directory (`~/vmware/name_of_the_vm`)

  ```bash
  # Create a working directory to pack the files and change into it
  mkdir box_to_package
  cd box_to_package
  
  # Create the "metadata.json" file used by Vagrant itself to specify it will require vmware_desktop provider to lauch new boxes out of this package
  cat << EOF > ./metadata.json
  {
    "provider": "vmware_desktop"
  }
  EOF
  
  # Copy the files that makes the VM (nvram, vmsd, vmx, vmxf, and vmdk) into your current working directory to pack them together. In this case the name of the VM is centos8 and is located at ~/vmware/centos8/
  for VM_FILES in nvram vmsd vmx vmxf vmdk; \
  do cp -a ~/vmware/centos8/*$(echo $VM_FILES) ./; \
  done
  
  # Package the VM by creating a compressed tar file. In this case it will be compressed in the file named centos8-vmw.box
  tar cvzf centos8-vmw.box ./*
  ```

* Add the packaged box into the Vagrant list of available boxes

  ```bash
  # Add the compressed file packaged into the Vagrant's box inventory.
  vagrant box add centos8.box --name centos-stream-8
  
  vagrant box list
  # ...output should should look like the next line below
  # centos8-vmw (vmware_desktop, 0)
  
  
  ```



**Create a Vagrant Box Out of the Box Added to The List**

Get ready to create your first Vagrant box out of the image/box we just added to the list. 
The hard work has been done an from now on every time we need a VM, all we have to do is initialize a working directory and tell Vagrant to launch the new box (or multiple boxes if we wanted to).

* Create a working directory to launch the box with Vagrant and the copy the SSH keys into them.

* ```bash
  # The idea is to create a folder to keep everything organized in a directory that will hold the SSH key copied from the VM and to hold a Vagrant file which will contain the code to define the box to be launched
  mkdir -p ./my_vagrant_box/keys
  cd ./my_vagrant_box/keys
  
  # If the VM still on VMware Workstation/Fusion you turn on the VM and copy the SSH keys out of the VM into your ./my_vagrant_box/keys directory
  scp vagrant@<vm_ip_address>:/home/vagrant/.ssh/id_rsa* ./
  
  # or
  
  # Copy the VM SSH keys from whatever location you choose from before into the ./my_vagrant_box/keys directory
  cp /path/to/custom_vagrant_ssh_keys/* .
  ```

  

* Initializes the current directory to be a Vagrant environment by creating an initial [Vagrantfile](https://www.vagrantup.com/docs/vagrantfile/)

  ```bash
  # Initialize the Vagrant environment. This will generate a Vagrant file that holds the code to define the VM/box properties to be launched.
  vagrant init centos-stream-8
  
  # Configure the Vagrant file to tell where to find the SSH keys by adding
  # config.ssh.private_key_path = .keys/id_rsa.pub to Vagrant file
  sed -i '/config.vm.box = "centos-stream-8"/ a \\n \ # Path to the private key SSH into the guest machine"\n  config.ssh.private_key_path = ".\/keys\/id_rsa.pub"' Vagrantfile
  ```

* Create your first Vagrant box out of the image/box we just added to the list.

  ```bash
  vagrant up
  
  vagrant ssh
  ```

* It's done. Enjoy!



**Full Workflow Example**	

```bash
# After creating and preparing the VM named 'centos8-vmw' in VMware Workstation/Fusion. Be aware the name of the VM is case sensitive.

# Name of the VM: centos8
# Directory where VMware Workstation/Fusion is storing the VM: ~/vmware/centos8/
# Name of box as will appear for Vagrant: centos8-vmw
# Working directory for Vagrant environment: ~/vagrant_boxes
# Direcoty storing a copy of the VM's SSH keys: ~/vagrant_boxes/keys


# Creating a working directory to work with Vagrant envirnoment and store the VM's ssh keys. You can change the '~/vagrant_boxes/keys' directory to whatever you like.
mkdir -p ~/vagrant_boxes/keys
cd ~/vagrant_boxes/keys

# Set the IP of the VM in a variable
# Change this value to the actual IP address of your VM
VM_IP='192.168.0.130'

# Copy the SSH keys out of the VM into our secured key folder
scp vagrant@$VM_IP:/home/vagrant/.ssh/id_rsa* ~/vagrant_boxes/keys

# SSH into the VM and execute the poweroff command to turn the VM off
ssh root@$VM_IP 'poweroff'

# Optimize the VM disk
vmware-vdiskmanager -d ~/vmware/centos8/centos8.vmdk
vmware-vdiskmanager -k ~/vmware/centos8/centos8.vmdk

# Create a working directory to pack the files and change into it
mkdir -p /tmp/box_to_package
cd /tmp/box_to_package

# Create the "metadata.json" file used by Vagrant itself to specify it will require vmware_desktop provider to lauch new boxes out of this package
cat << EOF > ./metadata.json
{
  "provider": "vmware_desktop"
}
EOF

# Copy the files that makes the VM (nvram, vmsd, vmx, vmxf, and vmdk) into your current working directory to pack them together. In this case the name of the VM is centos8 and is located at ~/vmware/centos8/
VM_NAME='centos8'
for VM_FILES in nvram vmsd vmx vmxf vmdk; \
do cp -a ~/vmware/$VM_NAME/*$(echo $VM_FILES) /tmp/box_to_package; \
done

# Package the VM by creating a compressed tar file. In this case it will be compressed in a tar file named centos8-vmw.box
BOX_NAME='centos8-vmw.box'
cd /tmp/box_to_package
tar cvzf $BOX_NAME ./*

# Add the compressed tar file packaged into the Vagrant's box inventory.
# The name of the Vagrant box is the name as it will be listed by 'vagrant box list' command. In this case 'centos8-vmw'
VAGRANT_BOX_NAME='centos8-vmw'
vagrant box add $BOX_NAME --name $VAGRANT_BOX_NAME

vagrant box list
# ...output should should look like the next line below
# centos8-vmw (vmware_desktop, 0)

############# Start working on the Vagrant environment #############
# Working Directory: '~/vagrant_boxes'
cd ~/vagrant_boxes/


# Initialize the Vagrant environment. This will generate a Vagrant file that holds the code to define the VM/box properties to be launched.
vagrant init centos8-vmw

# Configure the Vagrant file to tell where to find the SSH keys by adding
# config.ssh.private_key_path = .keys/id_rsa.pub to Vagrant file
sed -i '/config.vm.box = \"/ a \\n \ # Path to the private key SSH into the guest machine"\n  config.ssh.private_key_path = ".\/keys\/id_rsa"' Vagrantfile

# Review the Vagrant file configuration that has been created as the result of the 'vagrant init' command
grep -v '#' Vagrantfile
# ... output shoudl look like the below ...
#
# Vagrant.configure("2") do |config|
#  config.vm.box = "centos8-vmw"
# end

# Make Vagrant to create your first box out the Vagrantfile configuration and SSH into the box
vagrant up
vagrant ssh

```



#### Example of Vagrantfile

This section contains few examples of Vagrantfile with options to define SSH parameters, like user, password, and/or SSH key file to use.


**Vagrantfile**

Example of a `Vagrantfile` configuration specifying an SSH key (instead of using Vagrant insecure keys, default method)

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "centos8-vmw"
  config.ssh.private_key_path = "./keys/id_rsa"
  #config.ssh.username = "vagrant"
  #config.ssh.password = "vagrant"
end
```



Example of a `Vagrantfile` configuration specifying an username and a password for SSH access, instead of using keys as default method.

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "centos8-vmw"
  #config.ssh.private_key_path = "./keys/id_rsa"
  config.ssh.username = "vagrant"
  config.ssh.password = "vagrant"
end
```

