---
layout: post
title: Basic configuration for a new server using Puppet and Vagrant
description: "Starter configuration for a new server"
#modified: 2013-10-18
tags: [puppet, vagrant, server, configuration management, linux, ubuntu]
#share: true
---

Recently, I read [My First 5 Minutes On A Server][1] and [Automating Development Environments with Vagrant and Puppet][2].

On the hacker news discussion of the former, someone mentioned maybe what's needed is a starting point with a configuration management tool. 

I've been meaning to find out more [Vagrant](http://www.vagrantup.com) and [Puppet](http://puppetlabs.com/puppet/puppet-open-source) for a while, so I thought this is a good time to dig in, learn both, and implement the "five minutes on a server" recommendations in Puppet (with some of my own twists).

[1]: http://plusbryan.com/my-first-5-minutes-on-a-server-or-essential-security-for-linux-servers
[2]: http://blog.kloudless.com/2013/07/01/automating-development-environments-with-vagrant-and-puppet

### Vagrant
Creating a configuration management script for a new server doesn't actually involve Vagrant. It's included here because it's an awesome tool to manage virtual machines. This gives you a great way to test out your CM scripts without using an actual server. You do test out your CM script changes before you roll it out to a production server, right?

[Installing Vagrant](http://www.vagrantup.com/downloads) is easy. If you are on OS X, I used [brew](http://brew.sh) to install Vagrant. Then, you have to install a VM provider. Easiest (i.e. cheapest) one to use is [VirtualBox](https://www.virtualbox.org/wiki/Downloads).

Once they are installed, start VirtualBox at least once as there is a configuration for default VM location. Pick a directory to store you Vagrant configurations (it wouldn't be a bad idea to source control this folder too). Then, run `vagrant init`. This will create an initial version of the configuration. Your VM fun starts here. After some tweaking, mine ended up looking like this: [Vagrantfile][].

Let me go through the sections in the file.

{% highlight ruby %}
  config.vm.box = "puppet-labs-ubuntu-server-12042-x64-vbox4210-nocm."
  config.vm.box_url = "http://puppet-vagrant-boxes.puppetlabs.com/ubuntu-server-12042-x64-vbox4210-nocm.box"
{% endhighlight %}
This pulls down the preconfigured base box from puppetlabs.com. This is the blank slate you start with. I used Ubuntu x64 12.04 LTS *aka* Precise Pangolin. You can see more base boxes [here](http://puppet-vagrant-boxes.puppetlabs.com) and [here](http://www.vagrantbox.es). I also picked the version without any CM software installed. I did this because the reasoning from the [second article](http://blog.kloudless.com/2013/07/01/automating-development-environments-with-vagrant-and-puppet) - if you install your own CM software, then you can easily keep it up to date without changing the base box frequently.

{% highlight ruby %}
  config.vm.define :web do |web|
    web.vm.hostname = "dev-www.yellowaxe.vm"
    web.vm.network :private_network, ip: "192.168.33.10"
    web.vm.network :forwarded_port, guest: 80, host: 9080
    web.vm.network :forwarded_port, guest: 443, host: 7043
  end
{% endhighlight %}
This section defines a VM and gives it a hostname, IP, and forward some ports. Pretty self-explanatory.

{% highlight ruby %}
  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", "1024"]
  end
{% endhighlight %}
This section modifies the VM using VirtualBox specific tool. You can do a lot more here. Currently, I just have it increase the memory to 1GB.

{% highlight ruby %}
  # Enable shell provisioning to bootstrap puppet
  config.vm.provision :shell, :path => "puppet-bootstrap.sh"
{% endhighlight %}
This defines the first provisioner for the VM. It takes care of installing the latest version of puppet. The script is essentially the same as the one used in the kloudless article. It looks like this: [puppet-bootstrap.sh][].

{% highlight ruby %}
  # Enable provisioning with Puppet stand alone.
  config.vm.provision :puppet do |puppet|
        puppet.manifests_path = "puppet/manifests"
        puppet.manifest_file  = "site.pp"
        puppet.module_path = "puppet/modules"
        puppet.options = "--verbose"
        #puppet.options = "--debug --verbose"
  end
{% endhighlight %}
This defines the second provisioner - Puppet. This is the one that gets the work done.

### Puppet

Before I jump into the puppet configuration. Here is the directory structure of my puppet files under the Vagrant directory. As you add more puppet modules, there will be more directories.

```
[~/VMs/Vagrant] % tree puppet/manifests puppet/modules/yellowaxe
puppet/manifests
└── site.pp
puppet/modules/yellowaxe
├── files
│   └── kal
│       └── authorized_keys
└── templates
    ├── 00logwatch.erb
    ├── sshd_config.erb
    └── sudoers.puppet.erb
```
`puppet/manifests` typically stores all your `*.pp` puppet files. This are the configuration files written in the puppet language. You can break up the `site.pp` files into multiple files, but the puppet way is to create modules.

`puppet/modules` stores your own and any third part modules. You can get an idea on the modules available by browsing around [the forge](http://forge.puppetlabs.com).

The one module I listed is what I created to corral my configurations. For simplicity in this article, I kept all puppet code in `site.pp`. In a proper module setup, I would move the various class definitions into their rightful places in the modules directory.

Here is the full [site.pp][].

For each section, I will provide a brief explanation on the puppet syntax.

```
# create a new stage before the main stage
stage { 'first': } -> Stage['main']
```
By default, Puppet only runs 1 stage (think of as a grouping of tasks) and it's the main stage. In my configuration, I'd like to run certain tasks before the main stage. So, to guarantee that, I created this new stage called *first*. The symbol `->` is a chaining operator. It means: The newly created stage do its thing before the main stage.

Now, everything is added to main stage by default. So I will point where I have to indicate a certain task needs to be added to the *first* stage.

```
class yellowaxe::upgrade-all-pacakges {
  # apt-get update && apt-get upgrade
  # start the process with an updated system
  # this should be run before everything else (main stage)
  exec { 'apt-update':
    command => '/usr/bin/apt-get update',
  } ->
  exec { 'apt-upgrade':
    command => '/usr/bin/apt-get -y upgrade',
  }
}
```
`class` here defines a container of puppet resources. This is just a definition (like how a class in C++, Java is a definition). No action is actually done until you "instantiate" the class later.

This is the type of things that will go into a module file.

The class just runs 2 commands, chained together.

```
class yellowaxe::update-motd {
  file { '/etc/motd':
    ensure => file,
    content => "Welcome!",
  }
}
```
I don't like to leak too much information in MOTD, so we keep it simple. This also introduces the *file* puppet resource. In this case, it ensures the resource specified (using the path in the title section) is present and is a file type. And, the content of file will be updated if necessary to match the inlined simple string.

```
class yellowaxe::timezone($timezone = 'US/Central') {
  # configure time zones
  file { '/etc/localtime':
    ensure => link,
    target => "/usr/share/zoneinfo/$timezone",
  }
  file { '/etc/timezone':
    content => "$timezone\n",
  }
}
```
You will want to keep your servers on the same timezone generally. This will do that. I put in a default of US Central timezone, because that's where I am.

```
class yellowaxe::ban-failed-logins {
  # block failed logins, default seems decent
  package { 'fail2ban':
    ensure => installed,
  }
}
```
Make sure the fail2ban package is installed. We will go with the default for now.

```
class yellowaxe::normal-user($username) {
  # creates normal users, add more defaults here
  user { "$username":
    ensure => present,
    managehome => true,
    password => '*',
  }
}
```
This class will create a normal user, and disable password usage. Home directory will be created if needed.

```
class yellowaxe::sudo-user($username) {
  # creates users with sudo access, add more defaults here
  user { "$username":
    ensure => present,
    managehome => true,
    password => '*',
    groups => ['sudo'],
  }
}
```
This class will create a user that can `sudo`.

```
class yellowaxe::users {
  # disable root password
  # we don't want root to be used at all
  user { 'root':
    password => '*',
  }
}
```
Disable root password. While it's an option to just change the root password to a really long and complicated one, there is no reason you ever need to use root if there are multiple sudo-able users. If it is something you must do in the console, there is always single user mode.

```
class yellowaxe::sudo-config {
  # setup special sudo permissions
  # this allows a user to sudo without password, for example
  file { '/etc/sudoers.d/puppet':
    ensure => file,
    content => template('yellowaxe/sudoers.puppet.erb'),
    owner => 'root',
    group => 'root',
    mode => '0440',
  }
}
```
In addition to `/etc/sudoers`, additional sudo configuration is included from `/etc/sudoers.d` directory. Here, we add an additional sudoer configuration. This can be tailored specifically to fit your needs.

This uses Puppet's `template` function to fill out an ERB template file. You can learn more about it [here](http://docs.puppetlabs.com/guides/templating.html)

```
class yellowaxe::push-pubic-keys($username) {
  # push public keys for a given user
  file { "/home/$username/.ssh":
    ensure => directory,
    owner => "$username",
    group => "$username",
    mode => '0700',
    require => User["$username"],
  }
  file { "/home/$username/.ssh/authorized_keys":
    ensure => file,
    source => "puppet:///modules/yellowaxe/$username/authorized_keys",
    owner => "$username",
    group => "$username",
    mode => '0600',
    require => File["/home/$username/.ssh"],
  }
}
```
Since all passwords are disabled, we will be using public/private keys with SSH. If you have users that already have public keys available, this allows you push them right now.

```
class yellowaxe::ssh {
  # locking down SSH
  # Only thing changed is to disable root login and disallow passwords
  package { 'openssh-server':
    ensure => installed,
  } ->
  file { 'sshd_config':
    path => '/etc/ssh/sshd_config',
    ensure  => file,
    content => template('yellowaxe/sshd_config.erb'),
    owner => 'root',
    group => 'root',
    mode => '0640',
  } ~>
  service { 'sshd':
    name => 'ssh',
    ensure => running,
    enable => true,
    hasstatus => true,
    hasrestart => true,
  }
}
```
This locks down SSH configuration according to your needs. The template sshd_config file just have root login and password login disabled. You can add more of course.

```
class yellowaxe::firewall-rules {
  $ufw_bin = '/usr/sbin/ufw'
  # setup firewall rules
  package { 'ufw':
    ensure => present,
  } ->
  exec { "$ufw_bin --force reset": } ->
  exec { "$ufw_bin default deny incoming": } ->
  exec { "$ufw_bin default allow outgoing": } ->
  exec { "$ufw_bin allow http/tcp": } ->
  exec { "$ufw_bin allow https/tcp": } ->
  exec { "$ufw_bin allow ssh/tcp": } ->
  exec { "$ufw_bin limit ssh/tcp": } ->
  exec { "$ufw_bin --force enable": }
}
```
`ufw` is super easy to use, this is series of commands to reset and configure the firewall using `ufw`. It places a connection rate limit on the ssh port as well.

```
class yellowaxe::enable-auto-security-updates {
  # defaults on ubuntu precise is already security only
  package { 'unattended-upgrades':
    ensure => installed,
  }
}
```
Unattended upgrades isn't suitable for all situations. For a small shop, it's much better to have it on than depending on someone to do securities upgrade manually.

```
class yellowaxe::logwatch {
  # configure log watcher to email us
  package { 'logwatch':
    ensure => installed,
  } ->
  file { '00logwatch':
    path => '/etc/cron.daily/00logwatch',
    ensure  => file,
    content => template('yellowaxe/00logwatch.erb'),
  }
}
```
Very basic tool to monitor logs daily. The template adds the command to mail the report to me.

```
class yellowaxe::zsh {
  # I like zsh
  package { 'zsh':
    ensure => installed,
  }
}
```
I do like zsh.

```
class yellowaxe::zsh-user ($username) {
  # override the given user with zsh
  User <| title == "$username" |> {
    shell => '/usr/bin/zsh',
    require => Package['zsh'],
  }
}
```
Since we have zsh, some user would prefer to have this instead of the default shell. This uses a special collector syntax to override the user resource configuration. This might not be the most "puppet" way to do this, but it's pretty straightforward for me to understand.

```
node default {
```
A node in puppet term is a server. The `node` keyword can specifies 1 or more multiple servers. `default` is a special keyword that will match against any unspecified servers. Since I have only 1 server right now, I went ahead with default.

```
  class { 'yellowaxe::upgrade-all-pacakges':
    stage => first
  }
```
This forces `upgrade-all-packages` to run in the *first* stage instead of the main stage.

```
  include yellowaxe::update-motd
  include yellowaxe::timezone
  include yellowaxe::ban-failed-logins
  include yellowaxe::users
  include yellowaxe::sudo-config
  include yellowaxe::ssh
  include yellowaxe::logwatch
  include yellowaxe::firewall-rules
  include yellowaxe::enable-auto-security-updates
  include yellowaxe::zsh
```
This is a simple way to apply (e.g. instantiates) the class

```
  class { 'yellowaxe::normal-user':
    username => 'app',
  }
```
In this case, normal-user is used, but supplied with a new user name as "app".

The rest of the file is more of the same and you can see some additional modules that I have commented out, but can be used depending on the purpose of the server.

### Putting it all together

With this file and the supporting templates in place, Vagrant can now use all the pieces to provision the server. This as simple as running `vagrant up`.

This Vagrant and Puppet setup allows you to easily stand up and tear down development servers as VMs for testing. When it comes to deploying a production server, you will be faced with a blank server and the sequence becomes:

* run `puppet-bootstrap.sh`
* `git clone <your_tested_puppet_configuration_repo>` or copy over the puppet configuration via other means
* then `puppet apply <path_to_site.pp>` (the default modules directory is /etc/puppet/modules)

This is a puppet standalone setup. Puppet can also operate in a master/agent configuration and will allow you to execute puppet configuration remotely. I don't see myself needing a master/agent setup anytime soon.

### Closing

Vagrant is so nice. Simple and easy to learn. If you know even just a little ruby, the configurations will be very clear to you. It's too bad I took this long to try it.

It was pretty easy to learn enough Puppet in a day to put this together. In the process, I definitely feel there are some dissonances to using Puppet for me.

* Creating a new language isn't easy. The Puppet DSL has several gotchas (class inheritance oddities). Would using straight ruby avoid this kind of issues?
* I do appreciate the fact that sysadmins don't need to be programmers use Puppet, but I wonder how far you can take that. Good sysadmins are very paranoid people, I would be pretty leery of using third party modules without fully understanding what it's doing. And that means you will need to read and possibly modify ruby code.
* Having to use the Puppet language for definitions and having to use ruby for extensions to the system result in making some simple things more difficult. For example, in the user resource, I would have wanted to randomly generate a password and assign it to the user. This can be done simply using `pwgen` and `mkpasswd`, but there isn't a way for me to capture output of a command as a variable. I would have to write a custom function in ruby.

I am curious to see how [Chef](http://docs.opscode.com) and [Ansible](http://www.ansibleworks.com) compare to Puppet. Maybe next time.

### Appendix of all files

* [Vagrantfile][]
* [puppet-bootstrap.sh][]
* [site.pp][]

[Vagrantfile]: {{site.url}}/media/puppet/Vagrantfile.txt
[puppet-bootstrap.sh]: {{site.url}}/media/puppet/puppet-bootstrap.sh.txt
[site.pp]: {{site.url}}/media/puppet/site.pp.txt

