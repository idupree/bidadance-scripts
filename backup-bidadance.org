#!/bin/bash
#
# Run this backup script on a machine that has ssh to
# bidadance.org and that you want to store backups on.
# Each backup is approximately 50 MB as of this writing;
# most of that space is used by Wordpress code and
# Wordpress addons and git history of them.

set -eux

_sshargs="root@bidadance.org"
# sshargs="-i /path/to/id_rsa root@bidadance.org"

_backup_dir="${HOME}/bidadance-org-backups"

_date="$(date '+%Y-%m-%d_%H%M%S')"
_files_backup="/root/bida_files_backup_${_date}.tar.gz"
_db_backup="/root/bida_wordpress_dump_${_date}.sql.gz"

mkdir -p "${_backup_dir}"

ssh $_sshargs '
set -eux
cd /
tar -czf '"${_files_backup}"' /srv/bida-wordpress/ /etc/apache2/
mysqldump --defaults-extra-file=/root/.my-wordpress.cnf --single-transaction bida_wordpress | gzip > '"${_db_backup}"'
'

scp $_sshargs:"${_files_backup}" "${_backup_dir}"
scp $_sshargs:"${_db_backup}" "${_backup_dir}"

