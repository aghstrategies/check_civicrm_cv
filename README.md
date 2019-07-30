# Check CiviCRM using cv for Icinga2

This is a check designed to be executed locally on the server.  See https://github.com/MegaphoneJon/check_civicrm for an alternative that is designed for remote monitoring.

|                      | check_civicrm                    | check_civicrm_cv             |
| -------------------- | -------------------------------- | ---------------------------- |
| Executed             | Anywhere                         | On the web server            |
| Dependencies         | PHP                              | PHP and cv, Drush, or WP-CLI |
| Credentials required | Site key and a contact's API key | None                         |

*Note: Drush and WP-CLI support are only lightly tested.  I just added them because the command is basically the same.*

## Arguments

| Argument                | Description                                       |
| ----------------------- | ------------------------------------------------- |
| **-p /path/to/civicrm** | the directory of the site (similar to `cv --cwd`) |
| -x /path/to/cv          | the path to the `cv`, `drush`, or `wp` executable |
| -h                      | hide hidden system check messages                 |
| -t 2                    | the severity threshold for messages to appear     |
| -c 4                    | the severity for a "critical" exit status         |
| -w 3                    | the severity for a "warning" exit status          |
| -D                      | use Drush instead of cv                           |
| -W                      | use WP-CLI instead of cv                          |

The only required argument is `-p`.

The severities used in CiviCRM are as defined in [PSR-3](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-3-logger-interface.md#3-psrlogloggerinterface):

| Severity | Label     |
| -------- | --------- |
| 7        | EMERGENCY |
| 6        | ALERT     |
| 5        | CRITICAL  |
| 4        | ERROR     |
| 3        | WARNING   |
| 2        | NOTICE    |
| 1        | INFO      |
| 0        | DEBUG     |

## Shell usage

To confirm that the check works, you can run the check command in the shell:

> ./check_civicrm_cv -p /path/to/civicrm

You ought to get something along the lines of:

```
WARNING: 2 messages
==== System Status ====
[WARNING] WARNING: MySQL utf8mb4 Support: Future versions of CiviCRM may require MySQL utf8mb4 support. It is recommended, though not yet required, to configure your MySQL server for utf8mb4 support. You will need the following MySQL server configuration: innodb_large_prefix=true innodb_file_format=barracuda innodb_file_per_table=true
[WARNING] WARNING: Extension Update Available: Stripe (com.drastikbydesign.stripe) version 5.3.2 is installed. Upgrade to version 5.4.1.
[OK] NOTICE: Directory Paths: Make them portable: Directories may use absolute paths, relative paths, or variables. Absolute paths are more difficult to maintain. To maximize portability, consider using a variable in each directory (eg &#34;[cms.root]&#34; or &#34;[civicrm.files]&#34;).
[OK] NOTICE: Resource URLs: Make them portable: Resource URLs may use absolute paths, relative paths, or variables. Absolute paths are more difficult to maintain. To maximize portability, consider using a variable in each URL (eg &#34;[cms.root]&#34; or &#34;[civicrm.files]&#34;).
[OK] NOTICE: CiviCRM Patch Available: The site is running 5.14.1. Additional patches are available:
5.14.2 (2019-06-29): Fix recent regressions involving sample data-set and dashboard activity feed (with ACLs).
```

If you get something `UNKNOWN`, try this:

> cv --cwd=/path/to/civicrm api System.check

or this:

> drush -r /path/to/civicrm civicrm-api System.check --out=json

or this:

> wp --path=/path/to/civicrm civicrm api System.check --out=json

That's the underlying command, and it should give you a more useful error.

## User and permissions

You might have problems running this as a user other than the Apache/Nginx user.  To solve this, use `visudo` to let the Icinga2 user (`nagios` in our example) sudo without a password on this script.

> visudo

```
# User privilege specification
nagios ALL=(ALL) NOPASSWD: /path/to/check_civicrm_cv
```

Then, in your command definition, set the command to sudo with the user set in the `httpd_user` variable.

```
object CheckCommand "check_civicrm_cv" {
        import "plugin-check-command"

        command = [ "sudo", "-u", "$httpd_user$", PluginDir + "/check_civicrm_cv" ]

        arguments = {
                "-p" = "$civi_root$"
                "-h" = "$h$"
                "-s" = "$t$"
                "-x" = "$x$"
                "-c" = "$c$"
                "-w" = "$w$"
        }
}
```

Finally, set the variables in your service definition:

```
vars.civi_root = config.site_root
vars.x = "/usr/local/bin/cv"
vars.httpd_user = "www-data"
```
