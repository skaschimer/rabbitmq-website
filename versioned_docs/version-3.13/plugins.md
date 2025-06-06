---
title: Plugins
displayed_sidebar: docsSidebar
---
<!--
Copyright (c) 2005-2025 Broadcom. All Rights Reserved. The term "Broadcom" refers to Broadcom Inc. and/or its subsidiaries.

All rights reserved. This program and the accompanying materials
are made available under the terms of the under the Apache License,
Version 2.0 (the "License”); you may not use this file except in compliance
with the License. You may obtain a copy of the License at

https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Plugins

## Overview {#overview}

This guide covers

 * [Plugin support](#overview) in RabbitMQ
 * How to [enable a plugin](#ways-to-enable-plugins) using [CLI tools](./cli)
 * [Plugin directories](#plugin-directories)
 * How to [preconfigure plugins](#enabled-plugins-file) on a node at deployment time
 * [Troubleshooting](#troubleshooting) of a number of common issues
 * [Plugin support tiers](#plugin-tiers)

and more.

[Plugin development](/plugin-development) is covered in a separate guide.


## The Basics {#basics}

RabbitMQ supports plugins. Plugins extend core broker functionality in a variety of ways: with support
for more protocols, system state [monitoring](./monitoring), additional AMQP 0-9-1 exchange types,
node [federation](./federation), and more. A number of features are implemented as plugins
that ship in the core distribution.

This guide covers the plugin mechanism and plugins that ship in the [latest release](/release-information) of the RabbitMQ distribution.
3rd party plugins can be installed separately. A set of [curated plugins](#plugin-tiers) is also available.

Plugins are activated when a node is started or at runtime when a [CLI tool](./cli)
is used. For a plugin to be activated at boot, it must be enabled. To enable a plugin, use
the [rabbitmq-plugins](./cli):

```bash
rabbitmq-plugins enable <plugin-name>
```

For example, to enable the Kubernetes [peer discovery](./cluster-formation) plugin:

```bash
rabbitmq-plugins enable rabbitmq_peer_discovery_k8s
```

And to disable a plugin, use:

```bash
rabbitmq-plugins disable <plugin-name>
```

For example, to disable the [`rabbitmq-top`](./memory-use#breakdown-top) plugin:

```bash
rabbitmq-plugins disable rabbitmq_top
```

A list of plugins available locally (in the node's [plugins directory](./relocate)) as well
as their status (enabled or disabled) can be obtained using `rabbitmq-plugins list`:

```bash
rabbitmq-plugins list
```

To list available plugins without the legend header (marker explanations at the top):

```bash
rabbitmq-plugins list -s
```

To list available plugins in JSON:

```bash
rabbitmq-plugins list --formatter=json
```


## Different Ways to Enable Plugins {#ways-to-enable-plugins}

The `rabbitmq-plugins` command enables or
disables plugins by contacting the running node to tell it to
start or stop plugins as needed. It is possible to contact an arbitrary
node `-n` option to specify a different node.

Having a node running before the plugins are enabled is not always practical
or operator-friendly. For those cases `rabbitmq-plugins`
provides an alternative way. If the `--offline` flag is specified,
the tool will not contact any nodes and instead will modify the file containing
the list of enabled plugins (appropriately named `enabled_plugins`) directly.
This option is often optimal for node provisioning automation.

The `enabled_plugins` file is usually [located](./relocate) in the node
data directory or under `/etc`, together with configuration files. The file contains
a list of plugin names ending with a dot. For example, when [rabbitmq_management](./management) and
[rabbitmq_shovel](./shovel) plugins are enabled,
the file contents will look like this:

```erlang
[rabbitmq_management,rabbitmq_management_agent,rabbitmq_shovel].
```

Note that dependencies of plugins are not listed. Plugins with correct dependency metadata
will specify their dependencies there and they will be enabled first at the time of
plugin activation. Unlike `rabbitmq-plugins disable` calls against a running node,
changes to the file require a node restart.

The file can be generated by deployment tools. This is another automation-friendly approach.
When a plugin must be disabled, it should be removed from the list and the node must be restarted.

For more information on `rabbitmq-plugins`,
consult [the manual
page](./man/rabbitmq-plugins.8).


## Plugin Directories {#plugin-directories}

RabbitMQ loads plugins from the local filesystem. Plugins are distributed as
archives (`.ez` files) with compiled code modules and metadata.
Since some plugins [ship with RabbitMQ](#plugin-tiers), every
node has at least one default plugin directory. The path varies between
package types and can be [overridden](./relocate) using the
`RABBITMQ_PLUGINS_DIR` [environment variable](./configure#customise-environment).
Please see [File and Directory Locations guide](./relocate) to learn about the default
value on various platforms.

:::tip
the plugin directory can be a list of paths separated by a colon (on Linux, MacOS, BSD)
or a semicolon (Windows with PowerShell)
:::

The built-in plugin directory is by definition version-independent: its contents will change
from release to release. So will its exact path (by default) which contains version number,
e.g. `/usr/lib/rabbitmq/lib/rabbitmq_server-3.11.6/plugins`. Because of this
automated installation of 3rd party plugins into this directory is harder and more error-prone,
and therefore not recommended.

To solve this problem, the plugin directory can be a list of paths separated by a colon (on Linux, MacOS, BSD)
or a semicolon (Windows with PowerShell):

<Tabs>
<TabItem value="bash" label="bash" default>
```bash
# Example rabbitmq-env.conf file that features a colon-separated list of plugin directories
PLUGINS_DIR="/usr/lib/rabbitmq/plugins:/usr/lib/rabbitmq/lib/rabbitmq_server-3.11.6/plugins"
```
</TabItem>
<TabItem value="PowerShell" label="PowerShell" default>
```PowerShell
# Example rabbitmq-env-conf.bat file that features a colon-separated list of plugin directories
PLUGINS_DIR="C:\Example\RabbitMQ\plugins;C:\Example\RabbitMQ\rabbitmq_server-3.11.6\plugins"
```
</TabItem>
</Tabs>

Plugin directory paths that don't have a version-specific component and are not updated
by RabbitMQ package installers during upgrades are optimal for 3rd party plugin installation.
Provisioning automation tools can rely on those directories to be stable and only managed
by them.

3rd party plugin directories will differ from platform to platform and installation method
to installation method. For example, `/usr/lib/rabbitmq/plugins` is a 3rd party plugin directory
path used by RabbitMQ [Debian packages](./install-debian).

Plugin directory can be located by executing the `rabbitmq-plugins directories` command on the host
with a running RabbitMQ node:

```bash
rabbitmq-plugins directories -s
# => Plugin archives directory: /path/to/rabbitmq/plugins
# => Plugin expansion directory: /path/to/node/node-plugins-expand
# => Enabled plugins file: /path/to/enabled_plugins
```

The first directory in the example above is the 3rd party plugin directory.
The second one contains plugins that ship with RabbitMQ and will change as
installed RabbitMQ version changes between upgrades.

### rabbitmq-plugins {#offline-mode}

When plugin directories are overridden using an environment variable, the same variable
must also be set identically for the local OS user that invokes CLI tools.

If this is not done, [`rabbitmq-plugins` in offline mode](./cli#offline-mode) will not
locate the correct directory.


### The Enabled Plugins File {#enabled-plugins-file}

The list of currently enabled plugins on a node is stored in a file.
The file is commonly known as the enabled plugins file. Depending on the package type
it is usually located under the `etc` directory or under the node's
data directory. Its path can be [overridden](./configure) using the `RABBITMQ_ENABLED_PLUGINS_FILE`
environment variable. As a user you don't usually have to think about that file as it is
managed by the node and `rabbitmq-plugins` (when used in `--offline` mode).

Deployment automation tools must make sure that the file is readable and writeable by the local RabbitMQ node.
In environments that need to preconfigure plugins the file can be machine-generated at deployment time.
The plugin names on the list are exactly the same as listed by `rabbitmq-plugins list`.

The file contents is an Erlang term file that contains a single list:

```erlang
[rabbitmq_management,rabbitmq_management_agent,rabbitmq_mqtt,rabbitmq_stomp].
```

Note that the trailing dot is significant and cannot be left out.

### Plugin Expansion (Extraction) {#plugin-expansion}

Not every plugin can be loaded from an archive `.ez` file.
For this reason RabbitMQ will extract plugin archives on boot into a separate
directory that is then added to its code path. This directory is known
as the expanded plugins directory. It is usually managed entirely by RabbitMQ
but if node directories are changed to non-standard ones, that directory will likely
need to be overridden, too. It can be done using the `RABBITMQ_PLUGINS_EXPAND_DIR`
[environment variable](./configure#customise-environment). The directory
must be readable and writable by the effective operating system user of the RabbitMQ node.


## Troubleshooting {#troubleshooting}

:::tip
A significant majority of 3rd-party plugin activation issues come down to
insufficient filesystem permissions
:::

If a 3rd party plugin was installed but cannot be found, the most likely reasons
are

 * Incorrect plugin directory
 * `rabbitmq-plugins` and the server use different plugin directories
 * `rabbitmq-plugins` and the server use different enable plugin file
 * The plugin doesn't declare a dependency on RabbitMQ core
 * Plugin version is incompatible with RabbitMQ core

### 3rd Party Plugin Not Found {#troubleshooting-plugin-not-found}

When a plugin is enabled but the server cannot locate it, it will report an error.
Since any plugin name can be given to `rabbitmq-plugins`, double check
the name:

```bash
# note the typo
rabbitmq-plugins enable rabbitmq_managemenr
# => Error:
# => {:plugins_not_found, [:rabbitmq_managemenr]}
```

Another common reason is that plugin directory the plugin archive (the `.ez` file)
was downloaded to doesn't match that of the server.

Plugin directory can be located by executing the `rabbitmq-plugins directories` command on the host
with a running RabbitMQ node:

```bash
rabbitmq-plugins directories -s
# => Plugin archives directory: /path/to/rabbitmq/plugins
# => Plugin expansion directory: /path/to/node/node-plugins-expand
# => Enabled plugins file: /path/to/enabled_plugins
```

The first directory in the example above is the 3rd party plugin directory.
The second one contains plugins that ship with RabbitMQ and will change as
installed RabbitMQ version changes between upgrades.

`which` and similar tools can be used to locate `rabbitmq-plugins` and
determine if it comes from the expected installation:

```bash
which rabbitmq-plugins
# => /path/to/rabbitmq/installation/sbin/rabbitmq-plugins
```

### Plugin Cannot be Enabled {#troubleshooting-enabled-plugins-file-mismatch}

In some environments, in particular development ones, `rabbitmq-plugins`
comes from a different installation than the running server node. This can be the case
when a node is installed using a [binary build package](./install-generic-unix)
but CLI tools come from the local package manager such as `apt` or Homebrew.

In that case CLI tools will have a different [enabled plugins file](#enabled-plugins-file)
from the server and the operation will fail with an error:

```bash
rabbitmq-plugins enable rabbitmq_top
Enabling plugins on node rabbit@warp10:
# =>  rabbitmq_top
# =>  The following plugins have been configured:
# =>    rabbitmq_management
# =>    rabbitmq_management_agent
# =>    rabbitmq_shovel
# =>    rabbitmq_shovel_management
# =>    rabbitmq_top
# =>    rabbitmq_web_dispatch
# =>  Applying plugin configuration to rabbit@warp10...
# =>  Error:
# =>  {:enabled_plugins_mismatch, '/path/to/installation1/etc/rabbitmq/enabled_plugins', '/path/to/installation2/etc/rabbitmq/enabled_plugins'}
```

The first path in the error above corresponds to the enabled plugins file used by `rabbitmq-plugins`, the second
one is that used by the target RabbitMQ node.

`rabbitmqctl environment` can be used to inspect effective enabled plugins file path
used by the server:

```bash
rabbitmq-plugins directories -s
# => Plugin archives directory: /path/to/rabbitmq/plugins
# => Plugin expansion directory: /path/to/node/node-plugins-expand
# => Enabled plugins file: /path/to/enabled_plugins
```

Other common reasons that prevent plugins from being enabled can include [plugin archive](#plugin-directories)
and/or [plugin expansion](#plugin-expansion)
directories permissions not having sufficient privileges for the effective user of the server node. In other words,
the node cannot use those directories to complete plugin activation and loading.

### CLI Commands From a Plugin are Not Discovered {#troubleshooting-cli-command-discovery}

When performing command discovery, CLI tools will consult the [Enabled Plugins File](#enabled-plugins-file) to determine
what plugins to scan for commands. If a plugin is not included into that file, e.g. because it was enabled implicitly as
a dependency, it won't be listed in the enabled plugins file and thus its CLI commands **will not be discovered**.


## Plugin Tiers {#plugin-tiers}

Plugins that ship with the RabbitMQ distributions are often referred
to as tier 1 plugins. Provided that a standard distribution package is
used they do not need to be [installed](./installing-plugins) but do need to be
enabled before they can be used.

In addition to the plugins bundled with the server, team RabbitMQ
offers binary downloads of curated plugins which have been
contributed by authors in the community. See the [community plugins page](/community-plugins) for
more details.

Even more plugins can be found on GitHub, GitLab, Bitbucket and similar
services.

## Tier 1 (Core) Plugins in Open Source RabbitMQ {#tier1-plugins}

The table below lists tier 1 (core) plugins that ship with RabbitMQ.

<table class="plugins">
  <thead>
    <tr>
      <th>Plugin name</th>
      <th>Description</th>
    </tr>
  </thead>

  <tbody>
  <tr>
    <th>rabbitmq_amqp1_0</th>
    <td>
      AMQP 1.0 protocol support. For 4.x (currently in development), this plugin has
      been integrated into the core and becomes a no-op.

      <ul>
        <li>
          <a href="https://github.com/rabbitmq/rabbitmq-server/blob/main/deps/rabbitmq_amqp1_0/README.md">README for this plugin</a>
        </li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_auth_backend_ldap</th>
    <td>
      Authentication and authorisation plugin using an external
      LDAP server.
      <ul>
        <li><a href="./ldap">Documentation for the LDAP
        plugin</a></li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_auth_backend_oauth2</th>
    <td>
      Authentication and authorisation plugin using OAuth 2.0 protocol.
      <ul>
        <li><a href="./oauth2">Documentation for the OAuth 2.0 plugin</a></li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_auth_backend_http</th>
    <td>
      Authentication and authorisation plugin that uses an external HTTP API.
      <ul>
        <li>
          <a href="https://github.com/rabbitmq/rabbitmq-server/blob/main/deps/rabbitmq_auth_backend_http/README.md">README for this plugin</a>
        </li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_auth_mechanism_ssl</th>
    <td>
      Authentication mechanism plugin using SASL EXTERNAL to authenticate
      using TLS (x509) client certificates.
      <ul>
        <li>
          <a href="https://github.com/rabbitmq/rabbitmq-server/blob/main/deps/rabbitmq_auth_mechanism_ssl/README.md">README for this plugin</a>
        </li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_consistent_hash_exchange</th>
    <td>
      Consistent hashing exchange.
      <ul>
        <li>
          <a href="https://github.com/rabbitmq/rabbitmq-server/blob/main/deps/rabbitmq_consistent_hash_exchange/README.md">README for this plugin</a>
        </li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_federation</th>
    <td>
      Scalable messaging across WANs and administrative
      domains.
      <ul>
        <li><a href="./federation">Documentation for the
        federation plugin</a></li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_federation_management</th>
    <td>
      Shows federation status in the management API and UI. Only
      of use when using rabbitmq_federation in conjunction with
      <code>rabbitmq_management</code>. In a heterogeneous cluster this
      should be installed on the same nodes as rabbitmq_management.
    </td>
  </tr>

  <tr>
    <th>rabbitmq_jms_topic_exchange</th>
    <td>
      A special exchange type to be used with the <a href="https://github.com/rabbitmq/rabbitmq-jms-client">RabbitMQ JMS client</a>.
    </td>
  </tr>

  <tr>
    <th>rabbitmq_management</th>
    <td>
      A management / monitoring API over HTTP, along with a
      browser-based UI.
      <ul>
        <li><a href="./management">Documentation for the
        management plugin</a></li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_management_agent</th>
    <td>
      When installing the management plugin on <em>some</em>
      of the nodes in a cluster,
      <code>rabbitmq_management_agent</code> must be enabled on all
      on <em>all</em> cluster nodes nodes, otherwise stats for some nodes
      will not be collected.
    </td>
  </tr>

  <tr>
    <th>rabbitmq_mqtt</th>
    <td>
      MQTT 5 and 3.1.1 support.
      <ul>
        <li><a href="./mqtt">Documentation for the MQTT plugin</a></li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_prometheus</th>
    <td>
      Prometheus monitoring support.
      <ul>
        <li><a href="./prometheus">Documentation for monitoring with the Prometheus plugin</a></li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_sharding</th>
    <td>
       A plug-in for RabbitMQ that provides sharded queues. Sharding is
       performed by exchanges, that is, messages will be partitioned
       across "shard" queues by one exchange that we define as sharded.
      <ul>
        <li>
          <a href="https://github.com/rabbitmq/rabbitmq-server/blob/main/deps/rabbitmq_sharding/README.md">README for this plugin</a>
        </li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_shovel</th>
    <td>
      A plug-in for RabbitMQ that shovels messages from a queue on
      one broker to an exchange on another broker.
      <ul>
        <li>
          <a href="./shovel">Documentation for the Shovel plugin</a>
        </li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_shovel_management</th>
    <td>
      Shows <a href="./shovel">Shovel</a> status in the management API and UI.
      Only of use when using <code>rabbitmq_shovel</code> in
      conjunction with <code>rabbitmq_management</code>. In a
      heterogeneous cluster this should be installed on the same
      nodes as <a href="./plugins">RabbitMQ management plugin</a>.
    </td>
  </tr>

  <tr>
    <th>rabbitmq_stomp</th>
    <td>
      Provides <a href="http://stomp.github.io/stomp-specification-1.2.html">STOMP protocol</a> support in RabbitMQ.
      <ul>
        <li><a href="./stomp">Documentation for the STOMP plugin</a><br/></li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_tracing</th>
    <td>
      Adds message tracing to the management plugin. Logs
      messages from the <a href="./firehose">firehose</a> in
      a couple of formats.
    </td>
  </tr>

  <tr>
    <th>rabbitmq_trust_store</th>
    <td>
      Provides a client x509 certificate trust store.
      <ul>
        <li>
          <a href="https://github.com/rabbitmq/rabbitmq-server/blob/main/deps/rabbitmq_trust_store/README.md">README for this plugin</a>
        </li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_web_stomp</th>
    <td>
      STOMP-over-WebSockets: a bridge exposing <code>rabbitmq_stomp</code> to web
      browsers using WebSockets.
      <ul>
        <li><a href="./web-stomp">Documentation for the
        web-stomp plugin</a></li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_web_mqtt</th>
    <td>
      MQTT-over-WebSockets: a bridge exposing <a href="./mqtt">rabbitmq_mqtt</a> to Web
      browsers using WebSockets.
      <ul>
        <li><a href="./web-mqtt">Documentation for the
        web-mqtt plugin</a></li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_web_stomp_examples</th>
    <td>
      Adds some basic examples to
      <code>rabbitmq_web_stomp</code>: a simple "echo" service
      and a basic canvas-based collaboration tool.
      <ul>
        <li>
          <a href="https://github.com/rabbitmq/rabbitmq-server/blob/main/deps/rabbitmq_web_stomp_examples/README.md">README for this plugin</a>
        </li>
      </ul>
    </td>
  </tr>

  <tr>
    <th>rabbitmq_web_mqtt_examples</th>
    <td>
      Adds some basic examples to
      <code>rabbitmq_web_mqtt</code>: a simple "echo" service
      and a basic canvas-based collaboration tool.
      <ul>
        <li>
          <a href="https://github.com/rabbitmq/rabbitmq-server/blob/main/deps/rabbitmq_web_mqtt_examples/README.md">README for this plugin</a>
        </li>
      </ul>
    </td>
  </tr>
  </tbody>
</table>

## Additional Plugins in VMware Tanzu RabbitMQ® {#commercial-plugins}

The table below lists of plugins only available in [Tanuz RabbitMQ®](https://tanzu.vmware.com/rabbitmq).

<table class="plugins">
  <thead>
    <tr>
      <th>Plugin name</th>
      <th>Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <th>rabbitmq_schema_definition_sync</th>
      <td>
        Part of <a href="https://docs.vmware.com/en/VMware-RabbitMQ-for-Kubernetes/1/rmq/standby-replication.html">Warm Standby Replication</a>.
      </td>
    </tr>

    <tr>
      <th>rabbitmq_standby_replication</th>
      <td>
        Part of <a href="https://docs.vmware.com/en/VMware-RabbitMQ-for-Kubernetes/1/rmq/standby-replication.html">Warm Standby Replication</a>.
      </td>
    </tr>
  </tbody>
</table>

## Discontinued {#discontinued}

All plugins below have been **discontinued**. They no longer ship
with the RabbitMQ distribution and are no longer actively maintained by the RabbitMQ core team.

 * `rabbitmq_auth_backend_amqp`
 * `rabbitmq_management_visualiser`
