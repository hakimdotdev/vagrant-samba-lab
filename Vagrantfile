# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define "server1" do |server1|
    server1.vm.box = "ubuntu/focal64"
    server1.vm.hostname = "server1"
    server1.vm.network "private_network", ip: "192.168.50.10"
    server1.vm.network "public_network", adapter: 2
    server1.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      vb.cpus = 2
      vb.customize ["modifyvm", :id, "--cableconnected1", "on"]
    end
    
    require "optparse"

    # Parse the command-line arguments
    options = {}
    OptionParser.new do |opts|
      opts.banner = "Usage: vagrant up [options]"
      opts.on("-d DOMAIN", "--domain DOMAIN", "Domain name") { |v| options[:domain] = v }
      opts.on("-p PASSWORD", "--password PASSWORD", "Password") { |v| options[:password] = v }
    end.parse!

    # Set the default values for the domain and password variables
    domain = "mydomain.com"
    password = "mypassword"
    realm = domain
    # Extract the realm from the domain
    domain = domain.split(".")[0].upcase

    # Override the default values with the values from the command-line arguments, if provided
    domain = options[:domain] if options[:domain]
    password = options[:password] if options[:password]

    server1.vm.provision "shell", inline: <<-SHELL
      # Generate SSH keys for server1
      ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

      # Copy the public key of server1 to authorized_keys of server2
      # ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.50.11
	  
	    # Create the Netplan configuration
      echo '
      network:
        version: 2
        renderer: networkd
        ethernets:
          eth0:
            dhcp4: no
            addresses: [192.168.50.10/24]
            gateway4: 192.168.0.1
            nameservers:
              addresses: [192.168.50.10,192.168.50.11]
      ' > /etc/netplan/00-installer-config.yaml
      
      # Apply the configuration
      netplan apply
      apt-get update -y
      apt-get upgrade -y
      mv /etc/krb5.conf /etc/krb5.conf.bak
      apt-get -y install samba heimdal-clients smbclient winbind ntp ldb-tools
      mv /etc/samba/smb.conf{,.bu.orig}
      mv /etc/krb5.conf{,.bu.orig}
      mv /etc/ntp.conf{,.bu.orig}
      rm -f /run/samba/*.tdb
      rm -f /var/lib/samba/*.tdb
      rm -f /var/cache/samba/*.tdb
      rm -f /var/lib/

      # Run the samba-tool command using the values of the domain, rfc and password options
      samba-tool domain provision --use-rfc2307 --realm="#{domain}" --domain="#{realm}" --server-role=DC --dns-backend=SAMBA_INTERNAL --adminpass="#{password}"
      systemctl enable samba-ad-dc.service
      systemctl start samba-ad-dc.service
    end

  config.vm.define "server2" do |server2|
    server2.vm.box = "ubuntu/focal64"
    server2.vm.hostname = "server2"
    server2.vm.network "private_network", ip: "192.168.50.11"
    server2.vm.network "public_network", adapter: 2
    server2.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      vb.cpus = 2
      vb.customize ["modifyvm", :id, "--cableconnected1", "on"]
    end

    server2.vm.provision "shell", inline: <<-SHELL
      # Generate SSH keys for server2
      ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

      # Copy the public key of server2 to authorized_keys of server1
      ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.50.10
	  
	    # Create the Netplan configuration
      echo '
      network:
        version: 2
        renderer: networkd
        ethernets:
          eth0:
            dhcp4: no
            addresses: [192.168.50.11/24]
            gateway4: 192.168.50.1
            nameservers:
              addresses: [192.168.50.10,192.168.50.11]
      ' > /etc/netplan/00-installer-config.yaml
      
      # Apply the configuration
      netplan apply
      apt-get update -y
      apt-get upgrade -y
      mv /etc/krb5.conf /etc/krb5.conf.bak
      apt-get -y install samba heimdal-clients smbclient winbind ntp ldb-tools
      mv /etc/samba/smb.conf{,.bu.orig}
      mv /etc/krb5.conf{,.bu.orig}
      mv /etc/ntp.conf{,.bu.orig}
      rm -f /run/samba/*.tdb
      rm -f /var/lib/samba/*.tdb
      rm -f /var/cache/samba/*.tdb
      samba-tool domain join "#{domain}" DC -U Administrator --realm="#{realm}"
      cat <<EOF >/etc/netplan/00-installer-config.yaml
      network:
        version: 2
        renderer: networkd
        ethernets:
          eth0:
            dhcp4: no
            addresses: [192.168.50.11/24]
            gateway4: 192.168.50.1
            nameservers: {domain}
              addresses: [192.168.50.11,192.168.50.10]
      EOF
    SHELL
  end
end
