require 'getoptlong'
require 'yaml'

opts = GetoptLong.new(
        [ '--root-password', GetoptLong::REQUIRED_ARGUMENT ]
    )
opts.ordering=(GetoptLong::REQUIRE_ORDER)

root_password=nil
opts.each do |opt, arg|
  case opt
    when '--root-password'
      root_password = arg
  end
end
 
# Read YAML file with box details
servers_yaml = File.dirname(__FILE__) + '/servers.yaml'
servers  = YAML.load_file(servers_yaml)["servers"]
defaults = YAML.load_file(servers_yaml)["defaults"]
servers.each do |s|
    s.merge!(defaults){|k, spec, dflt| spec}
end
 
# Create boxes
Vagrant.configure(2) do |config|

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "postconfig.yaml"
    unless root_password.nil?
      ansible.raw_arguments = [
          "--extra-vars", "root_password=#{root_password}",
      ]
    end
    ansible.groups = {}
    servers.each do |server|
      if server.has_key?("roles")
        server["roles"].each do |r|
          if ansible.groups.has_key?(r)
            ansible.groups[r] << server["name"]
          else
            ansible.groups[r] = [ server["name"] ]
          end
        end
      end
    end
  end
 
  # Iterate through entries in YAML file
  servers.each do |servers|
    config.vm.define servers["name"] do |srv|
      svr = servers["name"]
      srv.vm.box = servers["box"]
      if servers.has_key?("network")
        servers["network"].each do |addr|
          addr.merge!(defaults["network"]){|k, spec, dflt| spec}
          ntype = addr[:type] || 'private_network'
          parms = addr.dup
          parms.delete(:type)
          parms.delete(:"default-gw")
          srv.vm.network(ntype, parms)
          if addr.has_key?(:"default-gw")
            gw = addr[:"default-gw"]
            srv.vm.provision "shell",
              run: "always",
              inline: "route add default gw #{gw} metric 120;
                       eval $(route -n |
                              awk '{ if ($1 == \"0.0.0.0\" && $2 != \"#{gw}\")
                                       print \"route del default gw \", $2 }');
                       /bin/true"
          end
        end
      end
      srv.vm.provider :virtualbox do |vb|
        vb.name   = servers["name"]
        vb.memory = servers["ram"]
        if servers.has_key?("cpus")
          vb.cpus = servers["cpus"]
        end
      end
    end
  end
end
