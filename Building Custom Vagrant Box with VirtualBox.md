### Building Custom CentOS 8 Vagrant Box Using VirtualBox as a Virtualization Provider



**Building a box**

First, we are going to create a virtual machine with VirtualBox, then setup the VM to meet Vagrant requirements, then package the VM into a Vagrant box.



**Prerequisite**

* VirtualBox
* Vagrant
* CentOS 8 ISO



**Prepare the VM**

* Create the virtual machine in VirtualBox

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

* Install Guest tools



**Package the VM into a Vagrant box format**

In the system were the VirtualBox is installed, powering the VM, were Vagrant is also installed, proceed with the following steps to pack the VM into a Vagrant box format: 

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
  
* Package the box

  ```bash
  # Create a working directory to pack the files and change into it
  mkdir box_to_package
  cd box_to_package
  
  vagrant package --base <virtualbox_vm_name> --output <name_of_my_custom_box>
  
  ls -lsh <name_of_my_custom_box>
  
  vagrant box add <name_of_my_custom_box> --name <my_custom_box_name>
  
  vagrant box list # You should see a list with the name of the custom box recently added
  
  ```
  
* Create a Vagrant box out of the custom box we just created, packaged, and added to the Vagrant list of local boxes and test it.

  ```bash
  vagrant init <my_custom_box_name>
  
  vagrant up
  
  vagrant ssh
  
  # Done!
  ```

  

  **Example**

  Example for a VM named `centos-stream-8`, created in VirtualBox (be aware the name of the VM is case sensitive, you must type the name of the VM exactly as it was named under VirtualBox):

  ```bash
  # After creating and preparing the VM named 'centos8-vmw' in VirtualBox. Be aware the name of the VM is case sensitive.
  
  mkdir centos-stream-8
  cd centos-stream-8
  
  vagrant package --base centos-stream-8 --output centos-stream-8-box
  
  vagrant box add centos-stream-8-box --name centos-stream-8-custom
  
  vagrant init centos-stream-8.box
  
  vagrant up
  
  vagrant ssh
  
  # Done!
  ```
  
  

#### Example of Vagrantfile

This section contains few examples of Vagrantfile with options to define SSH parameters, like user, password, and/or SSH key file to use.



**Vagrantfile**

Example of a `Vagrantfile` configuration specifying an SSH key (instead of using Vagrant insecure keys, default method)

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "centos8"
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



#### **How to copy a Vagrant box to another machine**

1. Copy the ssh key from `~/.vagrant.d/insecure_private_key` and append it to the same file on the other machine. This is necessary to be able to log into the VM later.
2. Check the VM name: `VBoxManage list vms`
3. Package the vagrant VM: `vagrant package --base vm-name --output /path/to/mybox.box`
4. Copy the `*.box` file and the Vagrantfile to the other machine
5. Modify the Vagrantfile: `config.vm.box_url= "/path/to/mybox.box"`
6. Run the VM: `vagrant up`

**Note:** Credits to https://stackoverflow.com/questions/19094024/is-there-any-way-to-clone-a-vagrant-box-that-is-already-installed

