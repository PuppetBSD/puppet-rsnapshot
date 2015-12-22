# rsnapshot

#### Table of Contents

1. [Overview](#overview)
2. [Module Description - What the module does and why it is useful](#module-description)
    * [Notes](#notes)
3. [Setup - The basics of getting started with rsnapshot](#setup)
    * [What rsnapshot affects](#what-rsnapshot-affects)
    * [Setup requirements](#setup-requirements)
    * [Getting started with rsnapshot](#getting-started)
4. [Configuration - options and additional functionality](#configuration)
    * [Examples](#examples)
    * [More Options](#more-options)
5. [Reference - An under-the-hood peek at what the module is doing and how](#reference)
    * [Classes](#classes)
    * [Functions](#functions)
    * [Parameters](parameters)
    * [Rsnapshot Configuration Parameters](#rsnapshot-configuration-variables)
5. [Limitations - OS compatibility, etc.](#limitations)
6. [Development - Guide for contributing to the module](#development)
7. [Editors](#editors)
8. [Contributors](#contributors)

## Overview

The rsnapshot module installs, configures and manages rsnapshot on a dedicated backup server.

## Module Description
The rsnapshot module installs, configures and manages rsnapshot on a dedicated backup server. It allows to set up a centralized Backup Server for all your nodes.
For the cron setup, the module will pick random time entries for the crons from an Array or a Range of time. For how to configure this, [please see below](#more-options)

### Notes
This module is best used with an ENC like hiera. It will make your config much easier to read and to maintain. Check the examples to see what I mean.

## Setup

### What rsnapshot affects

* This module will install the rsnapshot package on your system
* This module will manage the rsnapshot config on your system
* This module will manage cron entries for your configured nodes

### Setup Requirements

On CentOS Systems this module requires the stahnma-epel module. Also you will need to have rsync installed on all nodes to be backed up.
It will create repeatable random cron entries from a configurable timerange for all hosts.

### Getting Started

You will need to pass the nodenames to be backed up at least. 
This will pickup all defaults and add localhost to the backups:


```puppet
class { '::rsnapshot':
  hosts => {
    'localhost' => {},
  }
}
```

## Configuration
Here are some more elaborate examples of what you can do with this module.

### Examples
This will backup localhost with defaults. It will disable the default backup locations for example.com
and just backup '/var' for example.com.
```puppet
class { '::rsnapshot':
  hosts => {
    'localhost' => {},
    'example.com'    => {
      backup_defaults => false,
      backup          => {
        '/var/'       => './'
      }
    }
  }
}
```
The same in hiera:
```yaml
---
classes: rsnapshot
rsnapshot::hosts:
  localhost:
  example.com:
    backup_defaults: false
    backup:
      '/var/': './'
```



A more complete hiera example:
```yaml
---
classes: 
  - rsnapshot

# override default backup dirs for all hosts:
rsnapshot::default_backup:
    '/etc':         './'
    '/usr/local':   './'
    '/home':        './'

# configure hosts to be backed up
rsnapshot::hosts:
# pick all defaults for localhost
  localhost:
# add futher backups for node foo.example.com (additional to default_backup) and use a different snapshot root
  foo.example.com:
    backup:
      '/foo':       './'
      '/bar':       './'
      '/baz':       './misc'
    snapshot_root:  '/tmp/rsnapshot'
# all defaults
  foo1.example.com:
  foo2:
# disable default backup dirs and just backup /var for node bar1
# also set the minute to 0-10 for daily cron (note: this is not particularly useful, it's just meant to document the features)
# lastly set the range of hours to pick a random hour from (the cron for bar1 will have hour set to something between 1 and 5)
  bar1:
    backup_defaults: false
    backup:
      '/var': './var'
    cron:
      'daily':
        'minute': '0-10'
        'hour':   '1..5'


```



### More options
The defaults are pretty reasonable, I hope. However, you may override pretty much anything. Available parameters are discussed below. 

#### Specials
As mentioned, this module will generate random time entries for your hosts. The random number generator is hashed with hostname and backup_level, so the randomness will be repeatable per host.level. This is important so puppet won't override the crons with each run.
You may specify time ranges as follows:
  * default cron syntax (1-10, '*/5', 5)
  * an array with allowed values, for example, if you want the backup for a host to run between 1am and 5am, you would override the hours setting for the host in question.
in hiera this would look like: (Explanation see below)
```yaml
rsnapshot::hosts:
  example.com:
    cron:
      'daily':
        'minute': '1'
        'hour':   '1..5'
```
This will create the rsnapshot config using defaults from params.pp, but set the minute of the daily backup to '1' and the hour to something random between 1 and 5.
So it would look something like:
```
1 4 * * * foo daily
``` 
or maybe
```
1 2 * * * foo daily
```
## Reference

### Classes

#### Public Classes
* rsnapshot: Main class, includes all other classes.

####Private Classes

* rsnapshot::install: Handles the packages.
* rsnapshot::config: Handles configuration and cron files.
* rsnapshot::params: default values.

### Functions
####`assert_empty_hash`
Sets an empty value to a hash (we need this so a loop doesn't break if just a hostname is given to pick up all defaults.

####`pick_undef`
Like pick but returns undef values.

####`rand_from_array`
Takes an Integer, a String or an Array as input, and returns a random entry from the array (or just the String/Integer)

### Parameters

The following parameters are available in the `::rsnapshot` class:

####`$hosts`
Hash containing the hosts to be backed up and optional overrides per host
(Default: undef (do nothing when no host given))
####`$conf_d`
The place where the configs will be dropped 
(Default: /etc/rsnapshot (will be created if it doesn't exist))
####`$backup_user`
The user to run the backup scripts as 
(Default: root, also the user used for ssh connections, if you change this make sure you have proper key deployed and the user exists in the nodes to be backed up.)
####`$package_name`
(Default: rsnapshot)
####`$package_ensure`
(Default: present)
####`$cron_dir`
Directory to drop the cron files to. Crons will be created per host. 
(Default: /etc/cron.d)
####`$backup_levels`
Array containing the backup levels (hourly, daily, weekly, monthly)
Configure the backup_levels (valid per host and global, so you may either set: rsnapshot::backup_levels for all hosts or override default backup_levels for specific hosts)
(Default: [ 'daily', 'weekly', ] )
####`$backup_defaults`
Boolean. Backup default backup dirs or not.
(Default: true)

####`$config_default_backup`
The default backup directories. This will apply to all hosts unless you set [backup_defaults](#backup_defaults) = false
Default is:
```puppet
  $config_default_backup         = {
    '/etc'  => './',
    '/home' => './',
  }
```

####`$cron`
Hash. Set time ranges for different backup levels. Each item (minute, hour...) allows for cron notation, an array to pick a random time from and a range to pick a random time from.
The range notation is '$start..$end', so to pick a random hour from 8 pm to 2 am, you could set the hour of your desired backup level to 
`[ '20..23','0..2' ]`
Example:
```puppet
  $cron = {
    hourly     => {
      minute   => '0..59',
      hour     => [ '20..23','0..2' ],
    }
  }
```
Or in hiera:
- global override
```yaml
rsnapshot::cron:
  daily:
    minute: '20'
  weekly:
    minute: '20'
```
- per host override
```yaml
rsnapshot::hosts:
  webserver:
    daily:
      hour: [ '20..23','0..2' ]
    weekly:
      hour: [ '20..23','0..2' ]
```

Hash is of the form:
```puppet
$cron =>{
  daily => {
    minute => param,
    hour => param,
  }
  weekly => {
    minute => param,
    hour => param,
  }
  {...}
}
```

Default is:
```puppet
  $cron = {
    hourly     => {
      minute   => '0..59',  # random from 0 to 59
      hour     => '*',      # you could also do:   ['21..23','0..4','5'],
      monthday => '*',
      month    => '*',
      weekday  => '*',
    },
    daily      => {
      minute   => '0..10',      # random from 0 to 10
      hour     => '0..23',      # you could also do:   ['21..23','0..4','5'],
      monthday => '*',
      month    => '*',
      weekday  => '*',
    },
    weekly     => {
      minute   => '0..59',
      hour     => '0..23',      # you could also do:   ['21..23','0..4','5'],
      monthday => '*',
      month    => '*',
      weekday  => '0..6',
    },
    monthly    => {
      minute   => '0..59',
      hour     => '0..23',      # you could also do:   ['21..23','0..4','5'],
      monthday => '0..28',
      month    => '*',
      weekday  => '*',
    },
  }
```

####`$cron`
Hash. Set time ranges for different backup levels.
Hash is of the form:
```puppet
cron =>{
  daily => {
    minute => param,
    hour => param,
  }
  weekly => {
    minute => param,
    hour => param,
  }
  {...}
}
```
(Default:
```puppet
  $cron = {
    hourly     => {
      minute   => '0..59',
      hour     => '*',
      monthday => '*',
      month    => '*',
      weekday  => '*',
    },
    daily      => {
      minute   => '0..59',
      hour     => '0..23',
      monthday => '*',
      month    => '*',
      weekday  => '*',
    },
    weekly     => {
      minute   => '0..59',
      hour     => '0..23',
      monthday => '*',
      month    => '*',
      weekday  => '0..6',
    },
    monthly    => {
      minute   => '0..59',
      hour     => '0..23',
      monthday => '0..28',
      month    => '*',
      weekday  => '*',
    },
  }
```

####`$snapshot_root`
global. the directory holding your backups.
(Default: /backup)
You will end up with a structure like:
```
/backup/
├── example.com
│   ├── daily.0
│   ├── daily.1
│   ├── daily.2
│   ├── daily.3
│   ├── weekly.0
│   ├── weekly.1
│   ├── weekly.2
│   └── weekly.3
└── localhost
    ├── daily.0
    ├── daily.1
    ├── daily.2
    └── weekly.0
```

####`$default_backup`
The default backup directories (may be set per host, even though there is not much sense in doing so)
Default is:
```puppet
default_backup => {
  '/etc'  => './',
  '/home' => './',
}
```
####`$interval`
How many backups of each level to keep.
Default is:
```puppet
  $interval               = {
    'daily'   => '7',
    'weekly'  => '4',
    'monthly' => '6',
  }
```

### rsnapshot configuration variables
Please read up on the following in the [rsnapshot manpage](http://linux.die.net/man/1/rsnapshot)

####`$config_version`
Default is:  '1.2'

####`$cmd_cp`
Default is:  '/bin/cp'

####`$cmd_rm`
Default is:  '/bin/rm'

####`$cmd_rsync`
Default is:  '/usr/bin/rsync'

####`$cmd_ssh`
Default is:  '/usr/bin/ssh'

####`$cmd_logger`
Default is:  '/usr/bin/logger'

####`$cmd_du`
Default is:  '/usr/bin/du'

####`$cmd_rsnapshot_diff`
Default is:  '/usr/bin/rsnapshot-diff'

####`$cmd_preexec`
Default is:  undef

####`$cmd_postexec`
Default is:  undef

####`$use_lvm`
Default is:  undef

####`$linux_lvm_cmd_lvcreate`
Default is:  undef # '/sbin/lvcreate'

####`$linux_lvm_cmd_lvremove`
Default is:  undef # '/sbin/lvremove'

####`$linux_lvm_cmd_mount`
Default is:  undef # '/sbin/mount'

####`$linux_lvm_cmd_umount`
Default is:  undef # '/sbin/umount'

####`$linux_lvm_snapshotsize`
Default is:  undef # '100M'

####`$linux_lvm_snapshotname`
Default is:  undef # 'rsnapshot'

####`$linux_lvm_vgpath`
Default is:  undef # '/dev'

####`$linux_lvm_mountpath`
Default is:  undef # '/mount/rsnapshot'

####`$logpath`
Default is:  '/var/log/rsnapshot'

####`$logfile`
unused, we are logging to $logpath/$host.log
Default is:  '/var/log/rsnapshot.log'

####`$lockpath`
Default is:  '/var/run/rsnapshot'

####`$snapshot_root`
Default is:  '/backup/'

####`$no_create_root`
Boolean: true or false
Default is:  undef

####`$verbose`
Default is:  '2'

####`$loglevel`
Default is:  '4'

####`$stop_on_stale_lockfile`
Boolean: true or false
Default is:  undef

####`$rsync_short_args`
Default is:  '-az'

####`$rsync_long_args`
rsync defaults are: --delete --numeric-ids --relative --delete-excluded 
Default is:  undef

####`$ssh_args`
Default is:  undef

####`$du_args`
Default is:  undef

####`$one_fs`
Default is:  undef

####`$retain`
Default is:  { }

####`$include`
Default is:  []

####`$exclude`
Default is:  []

####`$include_file`
Default is:  undef

####`$exclude_file`
Other than this might suggest, the default behavior is to create an exclude file per host.
Default is:  undef

####`$link_dest`
Default is:  false

####`$sync_first`
Default is:  false

####`$rsync_numtries`
Default is:  1

####`$use_lazy_deletes`
Default is:  false

####`$backup_scripts`
Default is:  {}

## Limitations
Currently, this module support CentOS, Fedora, Ubuntu and Debian.

## Development
I have limited access to resources and time, so if you think this module is useful, like it, hate it, want to make it better or
want it off the face of the planet, feel free to get in touch with me.

## Editors
Norbert Varzariu (loomsen)

## Contributors
Please see the [list of contributors.](https://github.com/loomsen/puppet-bloonix_agent/graphs/contributors)

