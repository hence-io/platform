VAGRANTFILE_API_VERSION = "2"
Vagrant.require_version ">=1.7.2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

    $provision_timestamp = Time.now.to_i

    # --------------------------------------------------
    # Box Config
    # --------------------------------------------------

    # Load default vagrant config options
    require_relative 'config/default'

    # To update any of the above default options, add a custom.rb file that overrides any default values (found in 'default.rb') as required
    # --------------------------------------------------
    # EXAMPLE contents for custom.rb:
    # $rancher_ui_port = 3000
    # $vb_memory = 1024
    # $vb_cpus = 1
    # $rsync_folder_disabled = false
    # --------------------------------------------------

    CUSTOM_CONFIG = File.join(File.dirname(__FILE__), "config", "custom.rb")

    if File.exist?(CUSTOM_CONFIG)
        require CUSTOM_CONFIG
    end


    # --------------------------------------------------
    # Box Setup
    # --------------------------------------------------
    config.vm.box = $vm_box || "ubuntu/trusty64"
    config.vm.box_version = $vm_box_version || "20150609.0.10"

    if $vm_box == "rancherio/rancheros"
        config.vm.box_version = ">=0.3.3"
        require_relative 'vagrant_rancheros_guest_plugin.rb'

        config.vm.provider :virtualbox do |v|
            # On VirtualBox, we don't have guest additions or a functional vboxsf
            # in RancherOS, so tell Vagrant that so it can be smarter.
            v.check_guest_additions = false
            v.functional_vboxsf     = false
        end

        # plugin conflict
        if Vagrant.has_plugin?("vagrant-vbguest") then
            config.vbguest.auto_update = false
        end
    end

    # Disable synching current directory
    config.vm.synced_folder "./", "/vagrant", disabled: true

    # Set up the nodes
    (1..$number_of_nodes).each do |i|
        is_base_host = (i == 1)
        hostname = is_base_host ? $vm_name : "#{$vm_name}-%01d" % i

        config.vm.define hostname do |node|
            node.vm.provider :virtualbox do |v|
                v.name = hostname
                v.memory = $vm_memory || 2048
                v.cpus = $vm_cpus || 2
                v.customize ["modifyvm", :id, "--ioapic", "on"]

                # Set $vm_gui to `true` to enable gui mode so that you can choose to boot normally after an improper shutdown. This may help in the case of connection timeout errors:
                v.gui = $vm_gui || false
            end

            node.vm.hostname = hostname

            ip = "#{$private_ip_subnet}.#{i+99}"
            node.vm.network "private_network", ip: ip
            node.ssh.forward_agent = true

            # --------------------------------------------------
            # Port Forwarding
            # --------------------------------------------------
            if is_base_host then
                # Forward rancher_ui port
                node.vm.network "forwarded_port", guest: 8080, host: $rancher_ui_port, auto_correct: true
            end

            # Forward the docker daemon tcp.  Defaults to 2375.  Set to false in custom.rb to disable
            if $expose_docker_tcp
                node.vm.network "forwarded_port", guest: 2375, host: 2375, auto_correct: true
            end

            # Add any custom-configured exposed ports
            $forwarded_ports.each do |guest, host|
                node.vm.network "forwarded_port", guest: guest, host: host, auto_correct: true
            end

            # --------------------------------------------------
            # Folder Mounting
            # --------------------------------------------------
            node.vm.synced_folder "./projects", "/hence/projects", type: "nfs", :nfs_version => "3", :mount_options => ["actimeo=2"]
            node.vm.synced_folder "./mount", "/hence/mount", type: "rsync", rsync__exclude: ".git/", rsync__args: ["--verbose", "--archive", "--copy-links"]

            # Optionally, mount the User's home directory in the VM.  Defaults to false.
            if $share_home
                node.vm.synced_folder ENV['HOME'], ENV['HOME'], id: "home", type: "rsync", rsync__exclude: ".git/", rsync__args: ["--verbose", "--archive", "--delete", "--copy-links"]
            end

            # Optionally, mount arbitrary folders to the vm
            $shared_folders.each_with_index do |(host_folder, guest_folder), index|
                node.vm.synced_folder host_folder.to_s, guest_folder.to_s, id: "rancher-share%02d" % index, type: "rsync", rsync__exclude: ".git/", rsync__args: ["--verbose", "--archive", "--delete", "--copy-links"]
            end

            # --------------------------------------------------
            # Node Provisioning
            # --------------------------------------------------
            config.vm.provision :shell, path: "./scripts/setup/configure-folder-permissions.sh", :privileged => true
            config.vm.provision :shell, path: "./scripts/setup/configure-docker.sh", :privileged => true

            $rancher_server_image = "rancher/server:latest"
            $rancher_agent_image = "rancher/agent:latest"

            if is_base_host then
                config.vm.provision "docker" do |d|
                    d.run $rancher_server_image,
                        auto_assign_name: false,
                        daemonize: true,
                        restart: 'always',
                        args: "-p 8080:8080 --name rancher-server"
                end
            end

            config.vm.provision "docker" do |d|
                d.run $rancher_agent_image,
                    auto_assign_name: false,
                    daemonize: false,
                    restart: 'no',
                    args: "-e CATTLE_AGENT_IP=192.168.33.40 -e WAIT=true -v /var/run/docker.sock:/var/run/docker.sock --name rancher-agent-init",
                    cmd: "http://localhost:8080"
            end

            config.vm.provision :shell, :inline => "docker logs -f --since #{$provision_timestamp} rancher-agent-init", :privileged => true
            config.vm.provision :shell, :inline => "docker logs -f --since #{$provision_timestamp} rancher-agent", :privileged => true
        end
    end
end
