# ForemanAbrt

This plugin allows your Foreman instance to receive problem reports generated
on your hosts by [ABRT](https://github.com/abrt/abrt) (Automatic Bug Reporting
Tool). These reports can be inspected and eventually sent to the ABRT server
for analysis.

## Overview

1. Whenever a bug is caught by ABRT on the managed host, it is sent to the Smart
   proxy instead of being sent directly to the ABRT server.
2. The Smart proxy receives the report and stores it to the disk. Stored
   reports are then sent to Foreman every 30 minutes (by means of cron job).
   The proxy may optionally:
   - Forward the report to an ABRT server immediately after being received.
     Server's response is discarded.
   - Aggregate stored reports prior to sending them to the Foreman. Only one
     instance of set of similar reports from a host is sent, together with
     number of the reports in the set.
3. Foreman receives the aggregated report and stores it to the database. The
   reports can be inspected and forwarded ("sent for analysis") to the ABRT
   server. If the server responds with additional information about the report,
   such as links to bug trackers or suggested solutions, it is displayed
   alongside the report.

![foreman abrt workflow](/extra/foreman_abrt_workflow.png?raw=true)

### Network ports

Reports from managed hosts are sent to smart-proxy's https port which is 8443
for standalone Foreman installations and 9090 for Katello.

The proxy sends the reports using Foreman's API, which normally runs on
standard https (port 443).

## Installation

To be able to see ABRT problem reports in your Foreman instance, you need to
install the plugin itself, install ABRT plugin for your smart proxies and
configure your hosts to send the problem reports to their smart proxy.

Please note that the configuration is slightly different when using Foreman
stand-alone and when it is part of Katello. Any configuration specific to
**Katello** will be explicitly indicated.

Both plugins are available as RPMs in [Foreman YUM repositories](http://yum.theforeman.org/).

### Prerequisites

The plugins require both Foreman and smart-proxy to be version 1.7 or later.

The plugins have been tested on Fedora 19, RHEL 6 and RHEL 7. Versions of the
plugins compatible with Katello are `foreman_abrt-0.0.4` and
`smart_proxy_abrt-0.0.6`.

To have hosts automatically send ureports to Foreman, you need to have ABRT
2.1.11 or higher installed on them. RHEL 7 and Fedora 19 and higher satisfy
this. ABRT in RHEL6 does not have ureport support, however you can use an
[unofficial build](https://copr.fedoraproject.org/coprs/jfilak/abrt/) if you
wish to test this feature.

### Installing the Foreman plugin

To install the Foreman plugin, follow the [plugin installation
instructions](http://projects.theforeman.org/projects/foreman/wiki/How_to_Install_a_Plugin).

You need to install the `rubygem-foreman_abrt` package (or
`ruby193-rubygem-foreman_abrt` on RHEL/CentOS).

After installing the plugin, restart Foreman with `touch /usr/share/foreman/tmp/restart.txt`.

### Setting up smart proxies

Follow the [smart-proxy plugin installation
instructions](http://projects.theforeman.org/projects/foreman/wiki/How_to_Install_a_Smart-Proxy_Plugin).
You need to install the `rubygem-smart_proxy_abrt` package.

The plugin needs some configuration in order to work correctly.

- Ensure that `/etc/foreman-proxy/settings.d/abrt.yml` contains the following line:
  ```
  :enabled: true
  ```

- After installing and configuring the plugin, smart-proxy has to be restarted.
  ```
  service foreman-proxy restart
  ```

- Cron is used to transfer the captured problem reports to Foreman in batches.
  Ensure that the `smart-proxy-abrt-send` command is run periodically. The
  provided RPM package contains a cron snippet that runs the command every 30
  minutes.

### Configuring hosts to send problem reports to Foreman

This setup needs to be performed on every host that you wish to report its
crashes to Foreman. The host has to be managed by Puppet, i.e. host record must
exist in Foreman web interface.

- Make sure that ABRT is installed and running.
  ```
  ~# yum install abrt-cli
  ~# systemctl start abrtd
  ~# systemctl start abrt-ccpp
  ```

- Enable auto-reporting by running the following command:

  ```
  ~# abrt-auto-reporting enabled
  ```

- Configure ABRT reporting destination -
  `/etc/libreport/plugins/ureport.conf` should contain following:

  ```
  # URL of your foreman-proxy, with /abrt path.
  URL = https://smartproxy.tld:8443/abrt
  # Verify the server certificate.
  SSLVerify = yes
  # This asks puppet config for the path to the ceritificates. you can
  # explicitly provide path by using /path/to/cert:/path/to/key on the
  # right hand side.
  SSLClientAuth = puppet
  ```

  **Katello:** when using Foreman with Katello, the host needs to be registered
  (using subscription-manager) to your Katello instance. You can find the
  registration instructions on `https://foreman.tld/content_hosts/register`.
  The `/etc/libreport/plugins/ureport.conf` file should instead contain the
  following:

  ```
  # URL of your foreman-proxy, with /abrt path.
  URL = https://smartproxy.tld:9090/abrt
  # Verify the server certificate.
  SSLVerify = yes
  # Use the subscription management certificates to authenticate to the proxy.
  SSLClientAuth = /etc/pki/consumer/cert.pem:/etc/pki/consumer/key.pem
  ```

- Add the Puppet CA to the list of trusted certificate authorities. This is
  needed for verifying the validity of smart-proxy's certificate:

  ```
  ~# cp /var/lib/puppet/ssl/certs/ca.pem /etc/pki/ca-trust/source/anchors/
  ~# update-ca-trust
  ```

  **Katello:** when using Foreman with Katello, the subscription management
  certificate should be used instead:

  ```
  ~# cp /etc/rhsm/ca/katello-server-ca.pem /etc/pki/ca-trust/source/anchors/
  ~# update-ca-trust
  ```

### Verifying that the setup works

You can verify your setup by crashing something on your managed host. We have a
set of utilities in the Fedora/EPEL repository especially for this purpose:

```
~# yum -y install will-crash
~$ will_segfault
Will segfault.
Segmentation fault (core dumped)
```

After a couple of seconds, a new file should appear in
`/var/spool/foreman-proxy-abrt` on the smart-proxy host. The reports from
the smart-proxy are sent to the Foreman in batches every half an hour (by
default). This means that within half an hour you should be able to see the bug
report in the Foreman web interface. You can send the reports to Foreman
manually by running the `smart-proxy-abrt-send` command.

Please note that you need to enable the
[EPEL](https://fedoraproject.org/wiki/EPEL) repository if you are using RHEL in
order to install will-crash. If the will-crash package is not available, you
can try the following. Please make sure not to actually report this to `sleep`
maintainers, though:

```
~$ sleep 1d &
~$ kill -SEGV $!
```

### Testing aggregation

If you crash the same program twice (on one host) within the same period that
smart-proxy waits between forwarding the reports to foreman, then only one
report with count = 2 should appear in the web interface.

However, be careful about ABRT's rate limiting - if you crash a program
and then crash it again sooner that 20 seconds then the second crash is
simply ignored.

## Usage

The list of received problem reports can be accessed by clicking on *Problem
reports* link in the *Monitor* menu. To see detailed information for a report,
click on its reported date.

List of problem reports coming from a particular host is also displayed on the
page with the details about the host in the *Problem reports* tab on the left.

### Forwarding the report to the ABRT server

On the problem report details page you can forward the problem report to an
actual ABRT server by clicking the *Send for analysis* button. The ABRT server
may respond with some information it knows about the bug, such as the list of
URLs related to the bug (e.g. Bugzilla link) and list of possible solutions to
the problem that caused the bug to occur.

The forwarding functionality may have to be configured in *Abrt* tab of the
configuration screen (*Administer*->*Settings*).

## TODO

- Use puppet to configure managed hosts to send ureports to Foreman.

## Copyright

Copyright (c) 2014 Red Hat

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.

