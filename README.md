This is an Icinga2 plugin to monitor [Borg backups][borg]. While this
documentation does not cover how to use it with nagios, it can still be used
with it.

[borg]: https://borgbackup.readthedocs.io/en/stable/index.html

This plugin checks for 3 different things:

* last time of your most recent backup
* return codes of your automatic backup script
* repository consistency

# Dependencies

This plugin needs [borg][borg] to run properly.

# How to use it

## Return codes

This plugin checks the return code of your last borg run. Since you can not
extrapolate these return codes, you will need to log them somewhere when you
run your automatic borg scripts.

By default, the plugin looks for a file in `/var/log/borg/borg-rc.log` that
looks like this:

```
create: 0
prune: 0
```

To log the return codes, you can use something like this in you backup script:

```
#!/bin/sh
REPOSITORY=username@remoteserver.com:backup

borg create -v --stats                          \
    $REPOSITORY::'{hostname}-{now:%Y-%m-%d}'    \
    /home                                       \
    /var/www                                    \
    --exclude '/home/*/.cache'                  \
    --exclude /home/Ben/Music/Justin\ Bieber    \
    --exclude '*.pyc'
create_rc=$?

borg prune -v --list $REPOSITORY --prefix '{hostname}-' \
    --keep-daily=7 --keep-weekly=4 --keep-monthly=6
prune_rc=$?
  
echo "create: $create_rc\nprune: $prune_rc" > /var/log/borg/borg-rc.log
```

If you do not want to check the return codes on either `borg create` or `borg prune`,
you can pass the `-C` and `-P` arguments. More on this below.

## CLI options

<pre>
USAGE: 
  check_borg [-c 52] [-w 26] [-r repository] [-p repository_password] [-l return_code_log][-CPSh]
    -c Critical threshold in seconds (default: 187200 == 52 hours)
    -C don't check for borg create return code
    -h Show this help message
    -l return code log file (default: /var/log/borg/borg-rc.log)
    -p repository password
    -P don't check for borg prune return code
    -r repository
    -S run borg commands as super user (sudo)
    -w Warning threshold in seconds (default: 93600 == 26 hours)
</pre>

For example, you could call this plugin manually this way:

```
check_borg -r user@remoteserver:/data/borg/myrepository -p my_secret_password -l /var/log/borg/myrepository_rc.log
```

## Output

Depending on how the checks went, the plugin will output one of these messages:

* `OK`
* `UNKNOWN: return code log not found, not readable or incomplete`
* `WARNING: borg create reached its normal end, but there were warnings`
* `WARNING: borg prune reached its normal end, but there were warnings`
* `WARNING: borg check reached its normal end, but there were warnings`
* `WARNING: last complete backup was $time_since_last seconds ago. Warn is $WARN`
* `CRITICAL: borg create did not reach its normal end`
* `CRITICAL: borg prune did not reach its normal end`
* `CRITICAL: borg check did not reach its normal end `
* `CRITICAL: last complete backup was $time_since_last seconds ago. Crit is $CRIT`

## Icinga2 integration

To use `check_borg` with icinga2 you will need to add two things to your config.

First, we need to add a `CheckCommand` object. This will typically reside in
your `commands.conf` file in the `global-templates` zone:

```
object CheckCommand "borg" {
  command = [ PluginDir + "/check_borg" ] //constants.conf -> const PluginDir

  arguments = {
    "-c" = "$borg_critical_threshold$"
    "-w" = "$borg_warning_threshold$"
    "-l" = "$borg_rc_log$"
    "-p" = "$borg_password$"
    "-r" = "$borg_repository$"
    "-C" = {
      set_if = "$borg_dont_check_create_rc$"
      description = "Don't check for borg create return code."
    }
    "-P" = {
      set_if = "$borg_dont_check_purge_rc$"
		}
    "-S" = {
      set_if = "$borg_sudo$"
      description = "Run borg commands as super user (sudo)."
    }
  }
  timeout = 1h

  vars.borg_dont_check_create_rc = false
  vars.borg_dont_check_purge_rc = false
  vars.borg_sudo = false
```

You might want to monitor you first runs to check how much time the check takes
to run. `borg check` can take some time on large repositories and if you reach
the timeout icinga2 will kill the process and leave borg lock files in place.

Once that is done, you need to define a service. Here is an example of a service
you could declare:

```
object Service "borg" {
  import "generic-service"

  host_name = "my_server"
  check_command = "borg"
  check_interval = 1d
  vars.borg_repository = "username@remoteserver:/data/borg/myrepository"
	vars.borg_password = "my_secret_password"
}
```

### Remote repositories

Note that if you connect to a remote server via SSH with borg, the user running
icinga2 (`nagios` on Debian) will also need to be able to connect to this server.

You you can also bypass this issue by using the `-S` parameter and running all
borg commands during the check as root.

To give the `nagios` permission to run borg as root, you can add this in
`/etc/sudoers.d/20_nagios`:

```
nagios ALL = (root) NOPASSWD: /usr/bin/borg
```

### Downtimes

It might be a good idea to schedule a global downtime for icinga2 checks when
you backup your machine.

Please be aware that if you run `check_borg` while a backup is underway, at best
the check will fail because of the lock files and at worst it might affect the
backup procedures. Proceed with caution.

# Thanks

Most of the structure of this plugin comes from the great work Alexander Swen
and others did on the [check_puppet_agent][] plugin. Thank you!

[check_puppet_agent]: https://github.com/aswen/nagios-plugins.git
