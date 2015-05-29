# Capfile
## -*- mode: ruby -*-
## vi: set ft=ruby :

require "xp5k"
require "yaml"


# Load ./xp.conf file
#
XP5K::Config.load


# Initialize experiment
#
@xp = XP5K::XP.new(:logger => logger)
def xp; @xp; end


# Defaults configuration
#
XP5K::Config[:walltime]     ||= '1:00:00'
XP5K::Config[:user]         ||= ENV['USER']
XP5K::Config[:site]         ||= 'rennes'
XP5K::Config[:agents_count] ||= 2


# Constants
#
SSH_CONFIGFILE_OPT = XP5K::Config[:ssh_config].nil? ? "" : " -F " + XP5K::Config[:ssh_config]
SSH_CMD = "ssh -o ConnectTimeout=10" + SSH_CONFIGFILE_OPT


# Configure SSH for capistrano
#
set :gateway, XP5K::Config[:gateway] if XP5K::Config[:gateway]
set :ssh_config, XP5K::Config[:ssh_config] if XP5K::Config[:ssh_config]


# Define OAR jobs
#

# Manage OAR resources
resources = []
resources << %{nodes=#{1 + XP5K::Config[:agents_count]}}
resources << %{walltime=#{XP5K::Config[:walltime]}}

# Manage roles
roles = []
roles << XP5K::Role.new({ :name => 'puppetserver', :size => 1 })
roles << XP5K::Role.new({ :name => 'agents', :size => XP5K::Config[:agents_count] })

# Job description
job_description = {
  :resources => resources.join(","),
  :site      => XP5K::Config[:site],
  :queue     => XP5K::Config[:queue] || 'default',
  :types     => ["deploy"],
  :name      => "xp5k_puppet4",
  :roles     => roles,
  :command   => "sleep 186400"
}
job_description[:reservation] = XP5K::Config[:reservation] if not XP5K::Config[:reservation].nil?
xp.define_job(job_description)


# Define deployment on all nodes
#
xp.define_deployment({
  :site          => XP5K::Config[:site],
  :environment   => "jessie-x64-base",
  :jobs          => ["xp5k_puppet4"],
  :key           => File.read(XP5K::Config[:public_key]),
  :notifications => ["xmpp:#{XP5K::Config[:user]}@jabber.grid5000.fr"]
})


# Define Capistrano's roles
#
role :puppetserver do
  xp.role_with_name("puppetserver").servers
end

role :agents do
  xp.role_with_name("agents").servers
end


# Define the workflow
#
before :start, "oar:submit"
before :start, "kadeploy:submit"
before :start, "setup:repo"
before :start, "setup:agents"
before :start, "setup:puppetserver"


# Empty task for the `start` workflow
#
desc "Start the experiment"
task :start do
end


# Tasks for OAR job management
#
namespace :oar do
  desc "Submit OAR jobs"
  task :submit do
    xp.submit
    xp.wait_for_jobs
  end

  desc "Clean all running OAR jobs"
  task :clean do
    logger.debug "Clean all Grid'5000 running jobs..."
    xp.clean
  end

  desc "OAR jobs status"
  task :status do
    xp.status
  end
end


# Tasks for deployments management
#
namespace :kadeploy do
  desc "Submit kadeploy deployments"
  task :submit do
    xp.deploy
  end
end


# Setup Tasks
#
namespace :setup do
  desc "Install Puppet repository"
  task :repo, :roles => [:puppetserver, :agents] do
    set :user, 'root'
    run 'cd /tmp && http_proxy=http://proxy:3128 https_proxy=http://proxy:3128 wget http://apt.puppetlabs.com/puppetlabs-release-pc1-wheezy.deb'
    run 'dpkg -i /tmp/puppetlabs-release-pc1-wheezy.deb'
    run 'apt-get update'
  end

  desc "Install Puppet agent package"
  task :agents, :roles => [:puppetserver, :agents] do
    set :user, 'root'
    run 'apt-get -y install puppet-agent'
  end

  desc "Install Puppet server package"
  task :puppetserver, :roles => :puppetserver do
    set :user, 'root'
    run 'apt-get -y install puppetserver'
  end
end


# Tasks for open a shell on nodes
#
namespace :ssh do

  desc "ssh on the puppetserver node"
  task :puppetserver do
    fork_exec('ssh', SSH_CONFIGFILE_OPT.split(" "), 'root@' + serversForCapRoles('puppetserver').first)
  end

  desc "ssh on the first agent"
  task :agent do
    fork_exec('ssh', SSH_CONFIGFILE_OPT.split(" "), 'root@' + xp.role_with_name('agents').servers.first)
  end

end


# List servers from capistrano roles
#
def serversForCapRoles(roles)
  find_servers(:roles => roles).collect { |x| x.host }
end


# Fork the execution of a command. Used to execute ssh on deployed nodes.
#
def fork_exec(command, *args)
  # Remove empty args
  args.select! { |arg| arg != "" }
  args.flatten!
  pid = fork do
    Kernel.exec(command, *args)
  end
  Process.wait(pid)
end
