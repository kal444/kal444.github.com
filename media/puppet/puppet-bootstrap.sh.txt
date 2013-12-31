#!/usr/bin/env bash
set -e

if [ "$EUID" != "0" ] ; then
  echo "Script must be run as root." >&2
  exit 1
fi

if which puppet > /dev/null ; then
  echo "Puppet is already installed"
  exit 0
fi

echo "Installing Puppet repo for Ubuntu 12.04 LTS"
wget -qO /tmp/puppetlabs-release-precise.deb https://apt.puppetlabs.com/puppetlabs-release-precise.deb
dpkg -i /tmp/puppetlabs-release-precise.deb
rm /tmp/puppetlabs-release-precise.deb

apt-get update
apt-get -y install puppet
echo "Puppet installed!"

