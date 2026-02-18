# full: pigsty full-featured 4-node sandbox for HA-testing & tutorial & practices

Specs = [
  { "name" => "meta",   "ip" => "10.10.10.10", "cpu" => "8", "mem" => "16384", "image" => "bento/ubuntu-24.04" },
  { "name" => "node-1", "ip" => "10.10.10.11", "cpu" => "4", "mem" => "8192",  "image" => "bento/ubuntu-24.04" },
  { "name" => "node-2", "ip" => "10.10.10.12", "cpu" => "4", "mem" => "8192",  "image" => "bento/ubuntu-24.04" },
  { "name" => "node-3", "ip" => "10.10.10.13", "cpu" => "4", "mem" => "8192",  "image" => "bento/ubuntu-24.04" },
  { "name" => "node-4", "ip" => "10.10.10.14", "cpu" => "4", "mem" => "8192",  "image" => "bento/ubuntu-24.04" },
  { "name" => "node-5", "ip" => "10.10.10.15", "cpu" => "4", "mem" => "8192",  "image" => "bento/ubuntu-24.04" },


]

# This is the Vagrantfile template for libvirt (KVM) which require vagrant-libvirt plugin to work
# check https://vagrant-libvirt.github.io/vagrant-libvirt/ for details

# read ssh key from current user's ~/.ssh
ssh_prv_key = File.read(File.join(ENV['HOME'], '.ssh', 'id_rsa'))
ssh_pub_key = File.readlines(File.join(ENV['HOME'], '.ssh', 'id_rsa.pub')).first.strip

Vagrant.configure("2") do |config|
  config.ssh.insert_key = false
  config.vm.box_check_update = false
  config.vm.synced_folder ".", "/vagrant", disabled: true
 
  # SSH key and system configuration provisioning
  config.vm.provision "shell" do |s|
    s.inline = <<-SHELL
      if grep -sq "#{ssh_pub_key}" /home/vagrant/.ssh/authorized_keys; then
        echo "SSH keys already provisioned." ; exit 0;
      fi
      
      echo "SSH key provisioning..."
      sshd=/home/vagrant/.ssh
      mkdir -p ${sshd}
      touch ${sshd}/{authorized_keys,config}
      
      # Add SSH keys
      echo #{ssh_pub_key}   >> ${sshd}/authorized_keys
      echo #{ssh_pub_key}   >  ${sshd}/id_rsa.pub
      chmod 644 ${sshd}/id_rsa.pub
      echo "#{ssh_prv_key}" >  ${sshd}/id_rsa
      chmod 600 ${sshd}/id_rsa
      
      # Configure SSH client to not check host keys
      cat > ${sshd}/config <<'EOF'
StrictHostKeyChecking no
UserKnownHostsFile /dev/null
LogLevel ERROR

Host 10.10.10.* meta node-1 node-2 node-3
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
  User vagrant
  IdentityFile ~/.ssh/id_rsa
  IdentitiesOnly yes

Host *
  ServerAliveInterval 60
  ServerAliveCountMax 3
EOF
      chmod 600 ${sshd}/config
      
      # Add all nodes to /etc/hosts for easy resolution
      if ! grep -q "10.10.10.10 meta" /etc/hosts; then
        cat >> /etc/hosts <<'EOF'
10.10.10.10 meta
10.10.10.11 node-1
10.10.10.12 node-2
10.10.10.13 node-3
10.10.10.14 node-4
10.10.10.15 node-5

EOF
      fi
      
      # Fix SSH server configuration to prevent hanging connections
      echo "Configuring SSH server..."
      sed -i 's/^#*UseDNS.*/UseDNS no/' /etc/ssh/sshd_config
      sed -i 's/^#*GSSAPIAuthentication.*/GSSAPIAuthentication no/' /etc/ssh/sshd_config
      sed -i 's/^#*ListenAddress.*//' /etc/ssh/sshd_config
      sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
      sed -i 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
      sed -i 's/^#*ChallengeResponseAuthentication.*/ChallengeResponseAuthentication no/' /etc/ssh/sshd_config
      sed -i 's/^#*UsePAM.*/UsePAM no/' /etc/ssh/sshd_config
      
      # Ensure these settings are present
      if ! grep -q "^UseDNS no" /etc/ssh/sshd_config; then
        echo "UseDNS no" >> /etc/ssh/sshd_config
      fi
      if ! grep -q "^GSSAPIAuthentication no" /etc/ssh/sshd_config; then
        echo "GSSAPIAuthentication no" >> /etc/ssh/sshd_config
      fi
      if ! grep -q "^PasswordAuthentication no" /etc/ssh/sshd_config; then
        echo "PasswordAuthentication no" >> /etc/ssh/sshd_config
      fi
      if ! grep -q "^PubkeyAuthentication yes" /etc/ssh/sshd_config; then
        echo "PubkeyAuthentication yes" >> /etc/ssh/sshd_config
      fi
      if ! grep -q "^ChallengeResponseAuthentication no" /etc/ssh/sshd_config; then
        echo "ChallengeResponseAuthentication no" >> /etc/ssh/sshd_config
      fi
      if ! grep -q "^UsePAM no" /etc/ssh/sshd_config; then
        echo "UsePAM no" >> /etc/ssh/sshd_config
      fi
      
      # Ensure SSH listens on all interfaces (0.0.0.0)
      if ! grep -q "^ListenAddress 0.0.0.0" /etc/ssh/sshd_config; then
        echo "ListenAddress 0.0.0.0" >> /etc/ssh/sshd_config
      fi
      if ! grep -q "^ListenAddress ::" /etc/ssh/sshd_config; then
        echo "ListenAddress ::" >> /etc/ssh/sshd_config
      fi
      
      # Restart SSH service
      systemctl restart ssh
      
      # Disable UFW firewall (for development environment)
      if command -v ufw >/dev/null 2>&1; then
        ufw --force disable
        echo "UFW firewall disabled"
      fi
      
      # Flush all iptables rules to ensure no blocking
      echo "Flushing iptables rules..."
      iptables -F
      iptables -X
      iptables -t nat -F
      iptables -t nat -X
      iptables -t mangle -F
      iptables -t mangle -X
      iptables -P INPUT ACCEPT
      iptables -P FORWARD ACCEPT
      iptables -P OUTPUT ACCEPT
      
      # Save iptables rules
      if command -v netfilter-persistent >/dev/null 2>&1; then
        netfilter-persistent save
      fi
      
      # Set proper ownership
      chown -R vagrant:vagrant /home/vagrant
      
      echo "Provisioning complete!"
      exit 0
    SHELL
  end

  Specs.each_with_index do |spec, index|
    config.vm.define spec["name"] do |node|
      
      # Network configuration
      node.vm.network :public_network,
        libvirt__bridge: "bridge0",
        dev: "bridge0",
        mode: "bridge"
      
      node.vm.provider :libvirt do |libvirt|
        # Tell Vagrant to use your 'default' network instead of creating a new one
        libvirt.management_network_name = "default"
        # This MUST match the 'ip address' you defined in your XML (10.0.0.1)
        libvirt.management_network_address = "10.0.0.0/24"
        # Recommended: Keep Vagrant from deleting your 'default' network on destroy
        libvirt.management_network_keep = true
      end
      
      node.vm.box = spec["image"]
      node.vm.network "private_network", ip: spec["ip"]
      node.vm.hostname = spec["name"]
      
      # Storage configuration
      disk_size = spec["disk"] || 50
      node.vm.provider "libvirt" do |v|
        v.cpus   = spec["cpu"]
        v.memory = spec["mem"]
        
        if spec["name"].start_with?("minio")
          v.storage :file, size: '32G', device: 'vdb'
          v.storage :file, size: '32G', device: 'vdc'
          v.storage :file, size: '32G', device: 'vdd'
          v.storage :file, size: '32G', device: 'vde'
          v.storage :file, size: '32G', device: 'vdf'
          v.storage :file, size: '32G', device: 'vdg'
        else
          v.storage :file, size: "#{disk_size}G", device: 'vdb'
        end
      end

      # Disk provisioning for MinIO nodes
      if spec["name"].start_with?("minio")
        node.vm.provision "shell" do |s|
          s.inline = <<-SHELL
            mkdir -p /data1 /data2 /data3 /data4 /data5 /data6
            
            for dev in /dev/vdb /dev/vdc /dev/vdd /dev/vde; do
              if [ -b "${dev}" ] && ! blkid "${dev}" >/dev/null 2>&1; then
                if command -v mkfs.xfs >/dev/null 2>&1; then
                  mkfs.xfs -f "${dev}"
                elif command -v mkfs.ext4 >/dev/null 2>&1; then
                  mkfs.ext4 -F "${dev}"
                else
                  echo "[WARN] mkfs.xfs and mkfs.ext4 not found, skip formatting ${dev}"
                fi
              fi
            done
            
            if blkid /dev/vdb >/dev/null 2>&1 && ! grep -q '/dev/vdb' /etc/fstab; then
              echo "/dev/vdb /data1 auto defaults,noatime,nodiratime 0 0" >> /etc/fstab
            fi
            if blkid /dev/vdc >/dev/null 2>&1 && ! grep -q '/dev/vdc' /etc/fstab; then
              echo "/dev/vdc /data2 auto defaults,noatime,nodiratime 0 0" >> /etc/fstab
            fi
            if blkid /dev/vdd >/dev/null 2>&1 && ! grep -q '/dev/vdd' /etc/fstab; then
              echo "/dev/vdd /data3 auto defaults,noatime,nodiratime 0 0" >> /etc/fstab
            fi
            if blkid /dev/vde >/dev/null 2>&1 && ! grep -q '/dev/vde' /etc/fstab; then
              echo "/dev/vde /data4 auto defaults,noatime,nodiratime 0 0" >> /etc/fstab
            fi
            if blkid /dev/vde >/dev/null 2>&1 && ! grep -q '/dev/vde' /etc/fstab; then
              echo "/dev/vdf /data5 auto defaults,noatime,nodiratime 0 0" >> /etc/fstab
            fi
            if blkid /dev/vde >/dev/null 2>&1 && ! grep -q '/dev/vde' /etc/fstab; then
              echo "/dev/vdg /data6 auto defaults,noatime,nodiratime 0 0" >> /etc/fstab
            fi
            
            mount -a || true
          SHELL
        end
      
      # Disk provisioning for regular nodes
      else
        node.vm.provision "shell" do |s|
          s.inline = <<-SHELL
            if [ -b /dev/vdb ] && ! blkid /dev/vdb >/dev/null 2>&1; then
              if command -v mkfs.xfs >/dev/null 2>&1; then
                mkfs.xfs -f /dev/vdb
              elif command -v mkfs.ext4 >/dev/null 2>&1; then
                mkfs.ext4 -F /dev/vdb
              else
                echo "[WARN] mkfs.xfs and mkfs.ext4 not found, skip formatting /dev/vdb"
              fi
            fi
            
            mkdir -p /data
            
            if blkid /dev/vdb >/dev/null 2>&1 && ! grep -q '/dev/vdb' /etc/fstab; then
              echo "/dev/vdb /data auto defaults,noatime,nodiratime 0 0" >> /etc/fstab
            fi
            
            mount -a || true
          SHELL
        end
      end

      node.vm.provision "shell" do |s|
        s.inline = <<-SHELL
          set -e

          echo "Installing dependencies..."
          apt update -y
          apt install neovim -y
          apt install -y curl ca-certificates
          
          echo "Installing Pig CLI (with retry)..."

          for i in {1..5}; do
            if curl -fsSL https://repo.pigsty.io/pig | bash; then
              break
            fi
            echo "Retry $i failed... waiting 10s"
            sleep 10
          done

          # Verify pig installed
          if ! command -v pig >/dev/null 2>&1; then
            echo "Pig installation failed!"
            exit 1
          fi

          echo "Adding Pig repos..."
          pig repo add pigsty -u --region=default
          pig repo add pgdg -u --region=default
          pig repo add pgsql -u

          
          echo "Pig installed successfully"
          
          sleep 3
          pig repo add pgsql -u   
          pig repo add infra
          curl -fsSL https://repo.pigsty.io/key | sudo gpg --dearmor -o /etc/apt/keyrings/pigsty.gpg
          pig repo update
          
          pig repo set
        SHELL
      end
            


      # Always restart SSH after all networks are up to ensure it listens on all interfaces
      node.vm.provision "shell", run: "always" do |s|
        s.inline = <<-SHELL
          systemctl restart ssh
          echo "SSH restarted on all interfaces"
        SHELL
      end
      
    end
  end
end
