# create a new stage before the main stage
stage { 'first': } -> Stage['main']

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

class yellowaxe::update-motd {
  file { '/etc/motd':
    ensure => file,
    content => "Welcome!",
  }
}

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

class yellowaxe::ban-failed-logins {
  # block failed logins, default seems decent
  package { 'fail2ban':
    ensure => installed,
  }
}

class yellowaxe::normal-user($username) {
  # creates normal users, add more defaults here
  user { "$username":
    ensure => present,
    managehome => true,
    password => '*',
  }
}

class yellowaxe::sudo-user($username) {
  # creates users with sudo access, add more defaults here
  user { "$username":
    ensure => present,
    managehome => true,
    password => '*',
    groups => ['sudo'],
  }
}

class yellowaxe::users {
  # disable root password
  # we don't want root to be used at all
  user { 'root':
    password => '*',
  }
}

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

class yellowaxe::enable-auto-security-updates {
  # defaults on ubuntu precise is already security only
  package { 'unattended-upgrades':
    ensure => installed,
  }
}

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

class yellowaxe::zsh {
  # I like zsh
  package { 'zsh':
    ensure => installed,
  }
}

class yellowaxe::zsh-user ($username) {
  # override the given user with zsh
  User <| title == "$username" |> {
    shell => '/usr/bin/zsh',
    require => Package['zsh'],
  }
}

node default {
  class { 'yellowaxe::upgrade-all-pacakges':
    stage => first
  }
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
  class { 'yellowaxe::normal-user':
    username => 'app',
  }
  class { 'yellowaxe::sudo-user':
    username => 'kal',
  }
  class { 'yellowaxe::push-pubic-keys':
    username => 'kal',
  }
  class { 'yellowaxe::zsh-user':
    username => 'kal',
  }

  # additional modules to configure
  #
  # class { 'mongodb::globals':
  #   # use mongo's own repo instead of the distro's
  #   manage_package_repo => true,
  # } ->
  # class { 'mongodb::server':
  # }

  # class { 'redis':
  # }

  # class { 'ruby':
  #   gems_version => 'latest',
  # }

  # class { 'nodejs':
  # }

  # class { 'nginx':
  # }

  # class { 'rabbitmq':
  # }

  # haproxy

}

