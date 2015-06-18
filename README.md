collectd
========

Ansible role which installs CollectD.

The configuraton of the role is done in such way that it should not be necessary
to change the role for any kind of configuration. All can be done either by
changing role parameters or by declaring completely new configuration as a
variable. That makes this role absolutely universal. See the examples below for
more details.

Please report any issues or send PR.


Example
-------

```
---

# Example of default installation
- hosts: myhost1
  roles:
    - collectd

# Example of how to customize the default installation
- hosts: myhost2
  roles:
    - role: collectd
      # Collect metrics every 60 seconds
      collectd_global_options_default_interval: 60
      # Define additional global option
      collectd_global_options_custom:
        - options:
          - PIDFile: /var/run/collectd.pid

# Example of how to add additional plugin
- hosts: myhost3
  roles:
    - role: collectd
      collectd_plugins_custom:
        # Load df plugin
        - options:
          - LoadPlugin: df
        # Configure the df plugin
        - sections:
          - name: Plugin
            param: df
            content:
              # Options of the plugin
              - options:
                - ReportInodes: true
                - ValuesPercentage: true
        # Load threshold plugin
        - options:
          - LoadPlugin: threshold
        # Configure the threshold plugin
        - sections:
          - name: Plugin
            param: threshold
            content:
              # Add threshold for df plugin
              - sections:
                - name: Plugin
                  param: df
                  content:
                    - options:
                      - Instance: root
                    - sections:
                      - name: Type
                        param: percent_bytes
                        content:
                          - options:
                            - Persist: true
                            - PersistOK: true
                            - Instance: used
                            - WarningMax: 70
                            - FailureMax: 80

# Example of how to write completely new configuration
- hosts: myhost4
  roles:
    - role: collectd
      # Configure the collectd.conf file from cratch
      collectd_config:
        content:
          - options:
            - Hostname: myhost4
            - FQDNLookup: false
            - Interval: 10
            - Timeout: 2
            - ReadThreads: 5
            - AutoLoadPlugin: false
          - options:
            - LoadPlugin: syslog
          - sections:
            - name: Plugin
              param: syslog
              content:
                - options:
                  - LogLevel: info
          - options:
            - LoadPlugin: write_graphite
          - sections:
            - name: Plugin
              param: write_graphite
              content:
                - sections:
                  - name: Carbon
                    content:
                      - options:
                        - Host: 192.168.56.104
                        - Port: 2003
                        - Prefix: "collectd."
                        - StoreRates: false
                        - AlwaysAppendDS: false
                        - EscapeCharacter: _
          - options:
            - LoadPlugin: write_sensu
          - sections:
            - name: Plugin
              param: write_sensu
              content:
                - options:
                  - Tag: testtag1
                  - Attribute:
                    - collectd_sensu_test_param1
                    - foobar
                - sections:
                  - name: Node
                    param: localhost
                    content:
                      - options:
                        - Host: 127.0.0.1
                        - Port: 3030
                        - StoreRates: true
                        - AlwaysAppendDS: false
                        - Notifications: true
                        - MetricHandler: default
                        - NotificationHandler: default
                        - Separator: .
          - options:
            - LoadPlugin: df
          - sections:
            - name: Plugin
              param: df
              content:
                - options:
                  - ReportInodes: true
                  - ValuesPercentage: true
          - options:
            - LoadPlugin: cpu
            - LoadPlugin: disk
            - LoadPlugin: entropy
            - LoadPlugin: interface
            - LoadPlugin: irq
            - LoadPlugin: memory
            - LoadPlugin: processes
            - LoadPlugin: swap
            - LoadPlugin: users
            - LoadPlugin: vmem
          - options:
            - LoadPlugin: load
          - sections:
            - name: Plugin
              param: threshold
              content:
                - options:
                  - ReportRelative: true
          - options:
            - LoadPlugin: threshold
          - sections:
            - name: Plugin
              param: threshold
              content:
                - sections:
                  - name: Plugin
                    param: df
                    content:
                      - options:
                        - Instance: root
                      - sections:
                        - name: Type
                          param: percent_bytes
                          content:
                            - options:
                              - Persist: true
                              - PersistOK: true
                              - Instance: used
                              - WarningMax: 70
                              - FailureMax: 80

# Example of how to declare custom types
- hosts: myhost5
  roles:
    - role: collectd
      # Specify where to create the custom types.db file
      collectd_global_options_default_typedb_custom: /usr/share/collectd/types.db.custom
      # Specify the content of the file
      collectd_types:
        content:
          - options:
            # types with single datasource (value)
            - mybitrate: value:GAUGE:0:4294967295
            - mycounter: value:COUNTER:U:U
            # type with multiple data sources (rx, tx)
            - myifoctects:
              - rx:COUNTER:0:4294967295,
              - tx:COUNTER:0:4294967295
```

This role requires [Config Encoder
Macros](https://github.com/picotrading/config-encoder-macros) which must be
placed into the same directory as the playbook:

```
$ ls -1 *.yaml
site.yaml
$ git clone https://github.com/picotrading/config-encoder-macros.git ./templates/encoder
```


Role variables
--------------

List of variables used by the role:

```
# Package to be installed (you can force a specific version here)
collectd_pkg: collectd

# List of additional plugins (e.g. collectd-python)
collectd_pkg_plugins: []

# Default path to the main config file
collectd_config_path: /etc/collectd.conf

# Default list of custom types
collectd_types: {}


# Global options
collectd_global_options_default_hostname: "{{ ansible_hostname }}"
collectd_global_options_default_fqdnlookup: true
collectd_global_options_default_interval: 10
collectd_global_options_default_timeout: 2
collectd_global_options_default_readthreads: 5
collectd_global_options_default_autoloadplugin: false
collectd_global_options_default_typesdb: "{{
  [ collectd_global_options_default_typesdb_default ] + (
  [ collectd_global_options_default_typesdb_custom ]
  if collectd_global_options_default_typesdb_custom else [])
}}"

collectd_global_options_default:
  - options:
    - Hostname: "{{ collectd_global_options_default_hostname }}"
    - FQDNLookup: "{{ collectd_global_options_default_fqdnlookup }}"
    - Interval: "{{ collectd_global_options_default_interval }}"
    - Timeout:  "{{ collectd_global_options_default_timeout }}"
    - ReadThreads: "{{ collectd_global_options_default_readthreads }}"
    - AutoLoadPlugin: "{{ collectd_global_options_default_autoloadplugin }}"
    - TypesDB: "{{ pico_collectd_global_options_default_typesdb }}"


# Syslog plugin
collectd_plugins_syslog_loglevel: info

collectd_plugins_syslog:
  - options:
    - LoadPlugin: syslog
  - sections:
    - name: Plugin
      param: syslog
      content:
        - options:
          - LogLevel: "{{ collectd_plugins_syslog_loglevel }}"


# CPU plugin
collectd_plugins_cpu:
  - options:
    - LoadPlugin: cpu


# Interface plugin
collectd_plugins_interface:
  - options:
    - LoadPlugin: interface


# Load plugin
collectd_plugins_load:
  - options:
    - LoadPlugin: load


# Memory plugin
collectd_plugins_memory:
  - options:
    - LoadPlugin: memory


# Plugins definitions
collectd_plugins_default: "{{
  collectd_plugins_syslog +
  collectd_plugins_cpu +
  collectd_plugins_interface +
  collectd_plugins_load +
  collectd_plugins_memory
}}"


# Default list of custom global options
collectd_global_options_custom: []

# Default list of custom plugins
collectd_plugins_custom: []

collectd_config:
  content: "{{
    collectd_global_options_default +
    collectd_global_options_custom +
    collectd_plugins_default +
    collectd_plugins_custom
  }}"
```


Dependencies
------------

* [Config Encoder Macros](https://github.com/picotrading/config-encoder-macros)


License
-------

MIT


Author
------

Jiri Tyr
