# OWASP CRS - False Positive Report Plugin

## Description

This is a plugin that brings false positives tracking helper to CRS.

Plugin runs in phase 5 and watches all logs genereated by all rules and,
depending on filters configuration, sends e-mail notifications in case of
possible false positive (request does not need to be blocked for notification
to be generated).

Logs are read from memory using `WEBSERVER_ERROR_LOG` variable, so plugin does
not need any permissions to access log files on filesystem.

## Prerequisities

 * ModSecurity compiled with Lua support
 * LuaSocket library
 * plugin is able to catch only messages returned by rules with `log` action

## How to determine whether you have Lua support in ModSecurity

Most modern distro packages come with Lua support compiled in. If you are
unsure, or if you get odd error messages (e.g. `EOL found`) chances are you are
unlucky. To be really sure look for ModSecurity announce Lua support when
launching your web server:

```
... ModSecurity for Apache/2.9.5 (http://www.modsecurity.org/) configured.
... ModSecurity: APR compiled version="1.7.0"; loaded version="1.7.0"
... ModSecurity: PCRE compiled version="8.39 "; loaded version="8.39 2016-06-14"
... ModSecurity: LUA compiled version="Lua 5.3"
...
```

If this line is missing, then you are probably stuck without Lua. Check out the
documentation at [coreruleset.org](https://coreruleset.org/docs) to learn how to
get Lua support for your installation.

## LuaSocket library installation

LuaSocket library should be part of your linux distribution. Here is an example
of installation on Debian linux:  
`apt install lua-socket`

## Plugin installation

For full and up to date instructions for the different available plugin
installation methods, refer to [How to Install a Plugin](https://coreruleset.org/docs/4-about-plugins/4-1-plugins/#how-to-install-a-plugin)
in the official CRS documentation.

## Configuration

All settings can be done in file `plugins/false-positive-report-config.conf`.

### SMTP configuration

#### tx.false-positive-report-plugin_smtp_from

E-mail address which will be used as sender. You need to set this for plugin to
work.

Default value:

#### tx.false-positive-report-plugin_smtp_to

E-mail address to which notifications are send. You need to set this for plugin
to work.

Default value:

#### tx.false-positive-report-plugin_smtp_subject

Subject of e-mail message. These macros are supported:
 * <server_hostname> - hostname of server on which plugin runs on
 * <host_header> - content of `Host` HTTP header from request

Default value: <server_hostname>: False positive report from CRS

#### tx.false-positive-report-plugin_smtp_user

In case you want to use SMTP AUTH, fill in a username here. Otherwise, keep this
empty.

Default value:

#### tx.false-positive-report-plugin_smtp_password

In case you want to use SMTP AUTH, fill in a password here. Otherwise, keep this
empty.

Default value:

#### tx.false-positive-report-plugin_smtp_server

Hostname or IP address of SMTP server.

Default value: localhost

#### tx.false-positive-report-plugin_smtp_port

Port of SMTP server.

Default value: 25

### GeoIP configuration

GeoIP data can be get either using ModSecurity build-in GeoIP support or from an
external source (for example using data saved in environmental variables).

#### tx.false-positive-report-plugin_geoip_custom_lookup

This setting can be used to disable ModSecurity build-id GeoIP lookups and use
externally provided GeoIP data (for example mod_geoip2 / mod_maxminddb). See
setting `tx.false-positive-report-plugin_geoip_country_code` below.

Values:
 * 0 - disable custom GeoIP lookups and use ModSecurity built-in GeoIP lookups
 * 1 - enable custom GeoIP lookups

Default value: 0

#### tx.false-positive-report-plugin_geoip_country_code

Variable which holds GeoIP country code. Default value is suitable for
`mod_maxminddb`.

Default value: %{env.geoip_country_code}

### Filters

There can be thousands of logs generated every day so real false positives can
be easily overlooked. But no worry, filters comes to help! Using them you can
tell a plugin to send notifications only for a specific requests or simply
ignore logs matches a user-defined regexes.

#### tx.false-positive-report-plugin_filter_geoip

Space separated ISO codes of countries to watch for error messages. Logs from
requests which does not match this country list will be ignored. Keep this empty
to watch all countries.

Example: SK CZ

Default value:

#### tx.false-positive-report-plugin_filter_ignore_id

Space separated IDs of rules which you want to ignore.

Default value: 949110 959100 980130 980140

#### tx.false-positive-report-plugin_filter_ignore_request_method=CONNECT

Space separated HTTP request methods. Logs generated by requests which uses any
HTTP method from the list above will be ignored.

Example: CONNECT

Default value:

#### tx.false-positive-report-plugin_filter_ignore_request_uri_X=/.env

Array of regexes to match request URI. Logs generated by requests which matches
any request URI pattern from the list above will be ignored.

This setting is an array and you can use up to 100 patterns. For usage, see an
example below. There must be no gap in setting names numbering - all patterns
after a first gap will be ignored.

Example:
```
'tx.false-positive-report-plugin_filter_ignore_request_uri_1=/.env',\
'tx.false-positive-report-plugin_filter_ignore_request_uri_2=wp%-config',\
'tx.false-positive-report-plugin_filter_ignore_request_uri_3=/.git/config',\
```
See `Pattern matching` section below.

Default value:

#### tx.false-positive-report-plugin_filter_ignore_msg_1=

Array of regexes to match error message. Logs which matches any error message
pattern from the list above will be ignored - all patterns after a first gap
will be ignored.

This setting is an array so you can use up to 100 patterns. For usage, see an
example below. There must be no gap in setting names numbering.

See `Pattern matching` section below.

Example:
```
'tx.false-positive-report-plugin_filter_ignore_msg_1=Host header is a numeric IP address',\
'tx.false-positive-report-plugin_filter_ignore_msg_2=Found User%-Agent associated with security scanner',\
'tx.false-positive-report-plugin_filter_ignore_msg_3=Fake bot detected',\
```
Default value:

#### tx.false-positive-report-plugin_filter_ignore_pcre_limits_error

This setting can be used to ignore `PCRE limits exceeded` error messages.

Values:
 * 0 - don't ignore `PCRE limits exceeded` error messages
 * 1 - ignore `PCRE limits exceeded` error messages

Default value: 1

## Pattern matching

All patterns are matched using Lua patterns, which is a simplified version of
standard regular expressions. See Lua [documentation](https://www.lua.org/pil/20.2.html) for complete information.

These is one important thing to say: Lua patterns are using some, quite common,
special characters (for example `-` and `%`) as magic characters which has
special meaning. If you want to match such characters, you need to escape them
using `%` character. For example, this pattern can be used to match
`wp-config.php`: `wp%-config`

## Testing

After configuration, plugin should be tested, for example, using:  
```
curl https://yourdomain.tld/dump.sql
```

This will result in modsecurity sending an email to your configured email address, providing you with 
some basic data, an exclusion rule suggestion, relevant error logs etc.

## License

Copyright (c) 2022-2024 OWASP Core Rule Set project. All rights reserved.

The OWASP CRS and its official plugins are distributed
under Apache Software License (ASL) version 2. Please see the enclosed LICENSE
file for full details.
