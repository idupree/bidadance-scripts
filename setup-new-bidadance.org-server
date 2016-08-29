#!/bin/bash
# To use this script:
#
# Create a new Ubuntu 16.04+ VPS.
#
# ssh to this new server, and run this script there as root.
#
# Caveats:
#
# The old server still needs to be accessible
# by DNS at bidadance.org, or edit _old_server= below.
#
# When you ssh to the new server to run this script,
# you need to forward your ssh key, so this script
# can get the wordpress data from the old server.
#
# E.g., to forward ssh keys, use ssh -A,
# possibly preceded by this command if your system
# doesn't run an ssh-agent by default:
# eval `ssh-agent` && ssh-add /path/to/id_rsa

set -eux

# avoid /etc/hosts pointing "bidadance.org" to this very VM
_old_server="$(dig +short bidadance.org)"

_basic() {
  apt-get update
  apt-get upgrade -y
  apt-get install -y \
    vim emacs aptitude debconf-utils tmux htop ncdu curl rsync zsh \
    letsencrypt libapache2-mod-php apache2 \
    php-curl php-xmlrpc php-mysql php-intl php-gd php-cli php-json \
    php-readline php-mbstring
}

_get_mysql_pass() {
  local _user="$1"
  local _file="$2"
  if test -f "$_file"
  then
    local _pass="$(cat "$_file" | grep '^password=' | sed -E "s/password='(.*)'/\1/")"
  else
    local _pass="$(openssl rand -base64 16)"
    cat > "$_file" <<EOF
[client]
user=$_user
password='$_pass'
EOF
  fi
  printf '%s' "$_pass"
}

_mysql() {
  local _mysql_root_pass="$(_get_mysql_pass root /root/.my.cnf)"
  local _mysql_wp_pass="$(_get_mysql_pass bida_wordpress /root/.my-wordpress.cnf)"
  debconf-set-selections <<< "mysql-server mysql-server/root_password password $_mysql_root_pass"
  debconf-set-selections <<< "mysql-server mysql-server/root_password_again password $_mysql_root_pass"
  apt-get install -y mysql-server
  mysql_secure_installation --use-default
  mysql <<< "CREATE USER IF NOT EXISTS 'bida_wordpress'@'localhost' IDENTIFIED BY '$_mysql_wp_pass';"
  mysql <<< "CREATE DATABASE IF NOT EXISTS bida_wordpress"
  mysql <<< "GRANT ALL PRIVILEGES ON bida_wordpress.* TO 'bida_wordpress'@'localhost';"
}

#_copy_ssh_keys() {
#}

_copy_srv() {
  rsync -av --delete --chown=www-data:www-data root@"$_old_server":/srv/bida-wordpress/ /srv/bida-wordpress/
  rm -rf /srv/bida-wordpress/htdocs/wp-content/cache
  local _mysql_wp_pass="$(_get_mysql_pass bida_wordpress /root/.my-wordpress.cnf)"
  # `openssl rand -base64` won't generate any password characters
  # that are special in a sed replacement-string.
  sed -i "s~define *(.*DB_PASSWORD.*, .*);~define('DB_PASSWORD', '$_mysql_wp_pass');~" \
    /srv/bida-wordpress/wp-config.php
}

_copy_sql() {
  ssh root@"$_old_server" mysqldump \
    --defaults-extra-file=/root/.my-wordpress.cnf \
    --single-transaction \
    bida_wordpress \
    | gzip > /root/old_server_bida_wordpress.sql.gz
  zcat /root/old_server_bida_wordpress.sql.gz | \
    mysql --defaults-extra-file=/root/.my-wordpress.cnf bida_wordpress
}

_copy_apache() {
  rsync -av --delete root@"$_old_server":/etc/apache2/sites-available/ /etc/apache2/sites-available/
  rsync -av --delete root@"$_old_server":/etc/apache2/sites-enabled/ /etc/apache2/sites-enabled/
}

_apache() {
  a2enmod rewrite
  sudo systemctl restart apache2.service
}

_basic
_mysql
#_copy_ssh_keys
_copy_srv
_copy_sql
_copy_apache
_apache

