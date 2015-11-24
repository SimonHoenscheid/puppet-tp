---
layout: home
title: 'Tiny Puppet - Essential Applications Management'
subTitle: 'Yet Another Puppet Abstraction layer'
---

# Using Tiny Puppet


## Essential usage patterns

Install an application with default settings (package installed, service started)

    tp::install { 'redis': }

Configure the application main configuration file a custom erb template which uses data from a custom $options_hash:

    tp::conf { 'rsyslog':
      template     => 'site/rsyslog/rsyslog.conf.erb',
      options_hash => hiera('rsyslog::options_hash'),
    }

Populate any custom directory from a Git repository (it requires Puppet Labs' vcsrepo module):

    tp::dir { '/opt/apps/my_app':
      source      => 'https://git.example.42/apps/my_app/',
      vcsrepo     => 'git',
    }

Uninstall an application:

    tp::uninstall { 'redis': }


## Installation alternatives

Install custom packages (with the ```settings_hash``` argument we can override any application specific setting)

    tp::install { 'redis':
      settings_hash => {
        'package_name'     => 'my_redis',
        'config_file_path' => '/opt/etc/redis',
      },
    }

Use the ```tp::stdmod``` define to manage an application using stdmod compliant parameters.

Note that ```tp::stdmod``` is alternative to ```tp::install``` (both of them manage packages and services) and may be complementary to ```tp::conf``` (we can configure files with both).

    tp::stdmod { 'redis':
      config_file_template => 'site/redis/redis.conf',
    }


## Managing configurations

It's possible to manage files with different methods, for example directly providing its content:

    tp::conf { 'redis':
      content => 'my content is king',
    }

or providing a custom erb template (used as ```content => template($template)```):

    tp::conf { 'openssh::ssh_config':
      template    => 'site/openssh/ssh_config.erb',
    }

or using a custom epp template with Puppet code instead of Ruby (used as ```content => epp($epp)```):

    tp::conf { 'redis:
      epp   => 'site/redis/redis.conf.epp',
    }


also it's possible to provide the source to use, instead of managing it with the content argument:

    tp::conf { 'redis':
      source      => [ "puppet:///modules/site/redis/redis.conf-${::hostname}" ,
                       'puppet:///modules/site/redis/redis.conf' ] ,
    }


By default, configuration files managed by tp::conf automatically notify the service(s) and require the package(s) installed via tp::install. If we use tp::conf without a relevant tp::install define and have dependency cycle problems or references to non existing resources, we can disable these automatic relationships:

    tp::conf { 'bind':
      config_file_notify  => false,
      config_file_require => false,
    }

We can also set custom resource references to point to actual resources We declare in our manifests:

    tp::conf { 'bind':
      config_file_notify  => Service['bind9'],
      config_file_require => Package['bind9-server'],
    }


#### File paths conventions

Tp:conf has some conventions on the actual paths of the managed configuration files.

By default, if we just specify the application name, the file managed is the *main configuration file* of that application (in case this is not evident or may be questionable, check the data files for the actual value used for the settings key ```config_file_path```).

    # This manages /etc/ssh/sshd_config
    tp::conf { 'openssh':
      [...]
    }

If we specify a file name after the application name in the title, separated by ```::```, that file is placed in the *main configuration directory* (setting ```config_dir_path```):

    # This manages /etc/ssh/ssh_config
    tp::conf { 'openssh::ssh_config':
      [...]
    }

If we explicitly set a path, that path is used and the title is ignored (be sure, anyway, to refer to a supported application and is not duplicated in our catalog, in this way are automatically managed package dependencies and services notifications):

    # This manages /usr/local/bin/openssh_check
    tp::conf { 'openssh::ssh_check':
      path => '/usr/local/bin/openssh_check',
      [...]
    }

If we specify a ```base_dir``` and use a title with the format: ```application::file_name``` the file is created with the defined name in the indicated base directory (as , and if, configured in the settings):

    # Path is (in RedHat derivatives) /etc/httpd/conf.d/example42.com.conf
    tp::conf { 'apache::example42.com.conf':
      template => 'site/apache/example42.com.conf.erb',
      base_dir => 'conf', # Use the settings key: conf_dir_path
    }

There are different possible base_dir values, they may be defined according to the application. The most common ones are:

    base_dir param    Settings key       Description
    config            config_dir_path    The main configuration directory
    conf              conf_dir_path      A dir that contains fragments of configurations (usuallu /conf.d/)
    log               log_dir_path       Directory where are placed the application logs
    data              data_dir_path      Directory is placed application data


## Managing directories

Manage a whole configuration directory:

    tp::dir { 'redis':
      source      => 'puppet:///modules/site/redis/',
    }

Manage a specific directory type. Currently defined directories types are:
  - ```config``` : The application [main] configuration directory (Default value)
  - ```conf``` : A directory where we can place single configuration files (typically called ./conf.d )
  - ```data``` : Directory where application data resides
  - ```log``` : Dedicated directory for logs, if present

Note that some of these directory types might not be defined for every application.

    tp::dir { 'apache':
      base_dir => 'data',
      source   => 'puppet:///modules/site/apache/default_site',
    }

Clone a whole configuration directory from a Git repository (it requires Puppet Labs' vcsrepo module):

    tp::dir { 'redis':
      source      => 'https://git.example.42/puppet/redis/conf/',
      vcsrepo     => 'git',
    }

Populate any custom directory from a Subversion repository (it requires Puppet Labs' vcsrepo module):

    tp::dir { 'my_app': # The title is irrelevant, when path argument is used
      path        => '/opt/apps/my_app',
      source      => 'https://svn.example.42/apps/my_app/',
      vcsrepo     => 'svn',
    }

Provide a data directory (the default DocumentRoot, for apache) from a Git repository (it requires Puppet Labs' vcsrepo module) (TODO):

    tp::dir { 'apache':
      # base_dir is a tag that defines the type of directory for the specified application.
      # Default: config. Other possible dir types: 'data', 'log', 'confd', 'lib'
      # or any other name defined in the application data with a format like: ${base_dir}_dir_path
      base_dir    => 'data'
      source      => 'https://git.example.42/apps/my_app/',
      vcsrepo     => 'git',
    }


## Managing repositories

Currently Tiny Puppet supports applications' installation only via the OS native packaging system. In order to cope with software which may not be provided by default on an OS, TP provides the ```tp::repo``` define that manages YUM and APT repositories for RedHat and Debian based Linux distributions.

The data about a repository is managed as all the other data of Tiny Puppet. Find [here](https://github.com/example42/puppet-tp/blob/master/data/elasticsearch/osfamily/Debian.yaml) an example for managing Apt repositories and [here](https://github.com/example42/puppet-tp/blob/master/data/elasticsearch/osfamily/RedHat.yaml) one for Yum ones.

Generally we don't have to use directly the ```tp::repo``` defined, as, when the repository data is present, it's automatically added from the ```tp::install``` one.

If, for whatever reason, we don't want to automatically manage a repository for an application, we can set to ```false``` the ```auto_repo``` parameter, and, eventually we can manage the repository in a custom dependency class:

    tp::install { 'elasticsearch':
      auto_repo        => false,
      dependency_class => '::site::elasticseach::repo', # Possible alternative class to manage the repo
    }


## Usage with Hiera

We may find useful the ```create_resources``` defines that are feed, in the main ```tp``` class by special ```hiera_hash``` lookups that map all the available ```tp``` defines to hiera keys in this format ```tp::<define>_hash```.

Although such approach is very powerful (and totally optional) it's recommended not to abuse of it.

Tiny Puppet is intended to be used in modules like profiles, our data should map to parameters of such classes, but if we want to manage directly via Hiera some tp resources we have to include the main class:

    include tp

In the class are defined Hiera lookups (using hiera_hash so thy are recursive (and this may hurt a log when abusing) that expects parameters like the ones in the following sample in Yaml.

As an handy add-on, a ```create_resources``` is run also on the variables ```tp::packages```, ```tp::services```, ```tp::files``` to eventually manage the relevant Puppet resource types.

Not necessarily recommended, but useful to understand the usage basic patterns.

    ---
      tp::install_hash:
        memcache:
          ensure: present
        apache:
          ensure: present
        mysql
          ensure: present

      tp::conf_hash:
        apache:
          template: "site/apache/httpd.conf.erb"
        apache::mime.types:
          template: "site/apache/mime.types.erb"
        mysql:
          template: "site/mysql/my.cnf.erb"
          options_hash:


      tp::dir_hash:
        apache::certs:
          ensure: present
          path: "/etc/pki/ssl/"
          source: "puppet:///modules/site/certs/"
          recurse: true
          purge: true
        apache::courtesy_site:
          ensure: present
          path: "/var/www/courtesy_site"
          source: "https://git.site.com/www/courtesy_site"
          vcsrepo: git

      tp::puppi_hash:
        apache:
          ensure: present
        memcache:
          ensure: present
        php:
          ensure: present
        mysql:
          ensure: present

      tp::packages:
        wget:
          ensure: present
        zip:
          ensure: present
        curl:
          ensure: present

      tp::services:
        tuned:
          ensure: stopped
          enable: false
        NetworkManager:
          ensure: stopped
          enable: false