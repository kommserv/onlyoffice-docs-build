# OnlyOffice Docs (built from source using official build_tools)

I wrote this Ansible role to setup OnlyOffice Docs on an LXC container running
Ubuntu 14.04, in order to integrate it with one or more Nextcloud instances
running on another LXC container.

And I'm happy to report that it worked. I hit a few snags along the way,
but thanks to the very helpful OnlyOffice dev community I got just the help
I needed to work my way around them.

At first I set out to build [OnlyOffice from source](https://github.com/ONLYOFFICE/DocumentServer),
but I soon realised that this was not encouraged nor supported, nor could I find
any up-to-date guides on the web.
So I turned towards designing this Ansible role around the
[OnlyOffice `build_tools` repo](https://github.com/ONLYOFFICE/build_tools), which
is what this repo contains.

It is really the scripts inside the OnlyOffice `build_tools` repo that do
all the heavy lifting. This Ansible role is just a convenience in order for
me to integrate the provisioning of an OnlyOffice server into the rest of my
playbooks.

Although I wrote this role targeting an LXC container, it should of course
work just as well towards any Ubuntu 14.04 host.

If you decide to try it out, good luck! You will need it :-)

And if you have any suggestions for improvements, I would love to hear it!


> NOTE: building only the `server` component still fails, same videoplayer error as before,
> see `files/build-server-14.04-nodejs-10.24.1-npm-6.14.12-rpc.log` log file for details.



## Example playbook

```
- name: Build and configure OnlyOffice Docs
  hosts: onlyoffice_server

  tasks:
    # onlyoffice-docs-build role depends on:
    # fonts, nodejs, apache, redis, postgres
    - name: "OnlyOffice DocumentServer"
      import_role:
        name: onlyoffice-docs-build
      become: true
      tags: onlyoffice
```

Example `hosts.yml`:
```
onlyoffice_server:
  vars:
    ansible_python_interpreter: /usr/bin/python2
  hosts:
    lahore:
      ansible_host: lahore
      internal_ipv4: 10.252.116.226
```

Suggested `group_vars/onlyoffice_server.yml`:
```
---
psql_version: 9.3
redis_socket.path: "/var/run/redis/redis-server.sock"

#### NodeJS (important to use this specific npm version for OnlyOffice build to work)
nodejs_version: "10.24.1"
nodejs_use_deb: true
nodejs_use_nodesource: false
nodejs_use_tarball: false
nodejs_npm_packages:
  - name: npm
    version: "6.14.12"
  - name: grunt-cli
    version: "1.4.3"

#### Apache
apache_remove_default_vhost: true
APACHE_FQDN: onlyoffice.example.com
apache_packages:
  - apache2                     # Apache HTTP Server
  - apache2-utils               # Apache HTTP Server (utility programs for web servers)
  - libapache2-mod-fcgid        # FastCGI interface module for Apache 2
apache_mods_enabled:
  - mpm_event.load              # required for PHP-FPM
  - proxy.load                  # required for PHP-FPM
  - proxy_fcgi.load             # required for PHP-FPM
  - authn_core.load             # required for OnlyOffice
  - authz_core.load             # required for OnlyOffice
  - proxy.load                  # required for OnlyOffice
  - proxy_http.load             # required for OnlyOffice
  - proxy_wstunnel.load         # required for OnlyOffice
  - headers.load                # required for OnlyOffice
  - setenvif.load               # required for OnlyOffice
  - ssl.load                    # required for Apache SSL configuration
apache_mods_disabled:
  - mpm_prefork.load            # Apache can't work with multiple MPMs
  - mpm_prefork.conf            # so prefork needs to be explicitly disabled
  - "php{{ php_version }}.load" # related to pre-fork, not necessary with mpm_event

#### PHP 
# need to set the php_version variable since it's used by our apache and redis roles
# Ubuntu 14.04 uses PHP5
php_version: "5"
```

The other Ansible roles that this role depends on can be found
on [Codeberg.org/ansible](https://codeberg.org/ansible).
You could also replace them with your own roles or fulfill them in some
other way, of course.

Assuming you are running a recent Ubuntu release on your Ansible controller,
and assuming you are using Python3 as your Ansible Python interpreter,
you will likely get an error on connecting to the Ubuntu 14.04 target saying:
```
The following modules failed to execute: ansible.legacy.setup
Ansible requires a minimum of Python2 version 2.6 or Python3 version 3.5
```

That is because Python3 on Ubuntu 14.04 is horribly old, and so the mismatch
between the controller and the target is too large for Ansible to handle.
The easiest way around this is to make Ansible on the controller use Python2 instead,
by explicitly including `ansible_python_interpreter: /usr/bin/python2` in the host vars.



### Configuring and running the DocumentServer services

CAVEAT! At first inspection, I thought the provided JSON file `default.json` was
complete and that the provided files `development-linux.json` and
`production-linux.json` represented subsets of the default file.
But this turned out to not be the case, and means we should *merge* the default JSON
file with either production or development *before* applying our patches.

Diffing JSON files is not trivial. I found `vimdiff` to be excellent:
```
$ vimdiff <(jq --sort-keys . default.json) <(jq --sort-keys . development-linux.json)
```

Thanks to [Ashwin Jayaprakash for the tip](https://stackoverflow.com/questions/31930041/using-jq-or-alternative-command-line-tools-to-compare-json-files#comment69268125_37175540).

I found that *merging* JSON files is even less trivial.
In fact, I did it manually and include the result of merging `development-linux.json`
into `default.json` in `files/default-merged-dev.json`.
This is what we run against the tasks in this role using the Ansible `json_patch` plugin.

> NOTE: apart from `DocService`, you probably also want to run `FileConverter` and `SpellChecker`.
> https://helpcenter.onlyoffice.com/installation/docs-community-compile.aspx

I found no `SpellChecker` in the `out/` directory. Don't know what's up with that...
This role creates Upstart jobs for `DocService` and `FileConverter`.

The output from the DocService can be seen in the Upstart log at `/var/log/upstart/oodocservice.log`.

> In the `services.CoAuthoring.secret` block, for `browser.string`, `inbox.string`,
> `outbox.string`, and `session.string`, change **secret** to a secure, random
> alphanumeric string of your choice. 

I assume it should be the same secret string for all four, and that this is the
equivalent of `JWT_SECRET` for OODS Docker and such?

> In the `services.CoAuthoring.token` block, for `enable.browser`, 
> `enable.request.inbox`, `enable.request.outbox`, and `browser.secretFromInbox` 
> change the values from `false` to `true`.
> https://autoize.com/building-onlyoffice-document-server-from-source/

With build and configuration completed, https://onlyoffice.example.com/healthcheck should return the string **true**.



### Configuring Nextcloud to work with OnlyOffice

In Nextcloud's `config.php`, set
```
'allow_local_remote_servers' => true
```

Then enabled the app **OnlyOffice connector**, and filled out the
`OnlyOffice Docs address` and `Secret key` fields, and click `Save`.

That should be all!


## Refs

+ https://github.com/ONLYOFFICE/DocumentServer (OODS source code)
+ https://helpcenter.onlyoffice.com/installation/docs-community-install-ubuntu.aspx (the official install instructions)
+ https://github.com/ONLYOFFICE/document-server-proxy
+ https://helpcenter.onlyoffice.com/installation/docs-community-proxy.aspx


### Building from source

+ https://github.com/ONLYOFFICE/build_tools (OODS build tools for source code)
+ https://helpcenter.onlyoffice.com/installation/docs-community-compile.aspx (official build instructions)
+ https://autoize.com/building-onlyoffice-document-server-from-source/ (on Ubuntu 16.04, with PostgreSQL, mostly outdated)
+ https://www.howtoforge.com/how-to-compile-onlyoffice-document-server-from-source-code-on-ubuntu/ (simply repeats the official instructions)


### Using pre-built Linux DEB package

+ https://github.com/ONLYOFFICE/ansible-role-documentserver (Ansible role, OODS from Linux package on Debian/Ubuntu, with PostgreSQL)
+ https://git.coop/webarch/onlyoffice (Ansible role, OODS Linux packages on Debian, with PostgreSQL)
+ https://www.tecmint.com/install-onlyoffice-docs-on-debian-ubuntu/
+ https://tweenpath.net/install-onlyoffice-document-server-nginx-debian-10/
+ https://manjaro.site/how-to-install-onlyoffice-community-server-on-centos-7/ (configures OODS with MySQL on CentOS)
+ https://gist.github.com/tavinus/4cd108fa6a76c2a11da81a0e5c552bd0 (OODS on LXC Debian container with PostgreSQL)
+ https://sysadmin.exploinsights.com/index.php/2018/08/04/installing-onlyoffice-document-server-in-an-ubuntu-16-04-lxc-container/ (OODS on LXC Ubuntu container with PostgreSQL)
+ https://web.archive.org/web/20201108023712/https://sysadmin.exploinsights.com/index.php/2018/08/04/installing-onlyoffice-document-server-in-an-ubuntu-16-04-lxc-container/
+ https://ubuntu.com/tutorials/how-to-install-onlyoffice-server-on-ubuntu (how to install OODS using both DEB package and Docker container)
+ https://vitux.com/centos-onlyoffice-document-server/ (OODS on CentOS 7 with PostgreSQL)
+ https://www.atlantic.net/vps-hosting/how-to-install-onlyoffice-document-server-on-debian-10/ (OODS on Debian 10 with PostgreSQL)
+ https://www.linuxhowto.net/how-to-install-onlyoffice-docs-on-debian-and-ubuntu/

### Docker

+ https://help.nextcloud.com/t/strugging-integrating-onlyoffice-with-nextcloud-fpm-alpine-mariadb-redis-cron-lets-encrypt-docker-compose/66639/3
+ https://tech.oeru.org/installing-nextcloud-hub-onlyoffice-ubuntu-1804 (Nextcloud with MariaDB and OnlyOffice inside Docker (presumably with PostgreSQL in the container?))
+ https://www.linuxbabe.com/docker/onlyoffice-nextcloud-integration-docker
+ https://wiki.archlinux.org/title/Onlyoffice_Documentserver
+ https://programmerclick.com/article/15701925523/


### Snap

+ https://snapcraft.io/onlyoffice-ds



### On background

+ https://autoize.com/community-document-server-vs-onlyoffice-nextcloud-app/
+ https://helpcenter.onlyoffice.com/faq/server-opensource.aspx
+ https://github.com/ONLYOFFICE/CommunityServer (OnlyOffice Groups consists of many components, including: Jabber, Notify, Backup, Mail, WebStudio)
