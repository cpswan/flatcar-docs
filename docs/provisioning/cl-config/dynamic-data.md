---
title: Referencing dynamic data
weight: 40
aliases:
    - ../../container-linux-config-transpiler/doc/dynamic-data
    - ../../container-linux-config-transpiler/dynamic-data
---

## Overview

Sometimes it can be useful to refer to data in a Container Linux Config that isn't known until a machine boots, like its network address. This can be accomplished with [afterburn][afterburn] (previously called `coreos-metadata`). Afterburn is a very basic utility that fetches information about the current machine and makes it available for consumption. By making it a dependency of services which requires this information, systemd will ensure that coreos-metadata has successfully completed before starting these services. These services can then simply source the fetched information and let systemd perform the environment variable expansions.

While the `coreos-metadata.service` runs afterburn, it will not set the hostname. The hostname is set either through an OEM agent or for particular platforms through afterburn in the initramfs. If afterburn supports your platform and is not invoked in the initramfs by default, you can run it later to set the hostname (`--hostname=/etc/hostname`).

As of version 0.2.0, ct has support for making this easy for users. In specific sections of a config, users can enter in dynamic data between `{}`, and ct will handle enabling the coreos-metadata service and using the information it provides.

The available information varies by provider, and is expressed in different variables by coreos-metadata. If this feature is used a `--provider` flag must be passed to ct. Currently, the `etcd` and `flannel` sections are the only ones which support this feature.

[afterburn]: https://github.com/coreos/afterburn/

## Supported data by provider

This is the information available in each provider.

|                    | `HOSTNAME` | `PRIVATE_IPV4` | `PUBLIC_IPV4` | `PRIVATE_IPV6` | `PUBLIC_IPV6` |
|--------------------|------------|----------------|---------------|----------------|---------------|
| Azure              |            | ✓              | ✓             |                |               |
| Digital Ocean      | ✓          | ✓              | ✓             | ✓              | ✓             |
| EC2                | ✓          | ✓              | ✓             |                |               |
| GCE                | ✓          | ✓              | ✓             |                |               |
| Packet             | ✓          | ✓              | ✓             |                | ✓             |
| OpenStack-Metadata | ✓          | ✓              | ✓             |                |               |
| Vagrant-Virtualbox | ✓          | ✓              |               |                |               |

## Custom metadata providers

`ct` also supports custom metadata providers. To use the `custom` platform, create a coreos-metadata service unit to execute your own custom metadata fetcher. The custom metadata fetcher must write an environment file `/run/metadata/coreos` defining a `COREOS_CUSTOM_*` environment variable for every piece of dynamic data used in the specified Container Linux Config. The environment variables are the same as in the Container Linux Config, but prefixed with `COREOS_CUSTOM_`.

### Example

Assume `https://example.com/metadata-script.sh` is a script which communicates with a metadata service and then writes the following file to `/run/metadata/coreos`:
```
COREOS_CUSTOM_HOSTNAME=foobar
COREOS_CUSTOM_PRIVATE_IPV4=<The instance's private ipv4 address>
COREOS_CUSTOM_PUBLIC_IPV4=<The instance's public ipv4 address>
```

The following Container Linux Config downloads the metadata fetching script, replaces the ExecStart line in `coreos-metadata` service to use the script instead, and configures etcd using the metadata provided. Use the `--platform=custom` flag when transpiling.
```yaml
storage:
  files:
    - filesystem: "root"
      path: "/opt/get-metadata.sh"
      mode: 0755
      contents:
        remote:
          url: "https://example.com/metadata-script.sh"

systemd:
  units:
    - name: "coreos-metadata.service"
      contents: |
        [Unit]
        Description=Metadata agent
        After=nss-lookup.target
        After=network-online.target
        Wants=network-online.target
        [Service]
        Type=oneshot
        Restart=on-failure
        RemainAfterExit=yes
        ExecStart=/opt/get-metadata.sh

etcd:
  version:                     "3.0.15"
  name:                        "{HOSTNAME}"
  advertise_client_urls:       "http://{PRIVATE_IPV4}:2379"
  initial_advertise_peer_urls: "http://{PRIVATE_IPV4}:2380"
  listen_client_urls:          "http://0.0.0.0:2379"
  listen_peer_urls:            "http://{PRIVATE_IPV4}:2380"
  initial_cluster:             "{HOSTNAME}=http://{PRIVATE_IPV4}:2380"
```

You can find another example in the [VMware docs](../../installing/cloud/vmware.md).

## Behind the scenes

For a more in-depth walk through of how this feature works, let's look at the etcd example from the [examples document][examples].

```yaml
etcd:
  version:                     "3.0.15"
  name:                        "{HOSTNAME}"
  advertise_client_urls:       "http://{PRIVATE_IPV4}:2379"
  initial_advertise_peer_urls: "http://{PRIVATE_IPV4}:2380"
  listen_client_urls:          "http://0.0.0.0:2379"
  listen_peer_urls:            "http://{PRIVATE_IPV4}:2380"
  initial_cluster:             "{HOSTNAME}=http://{PRIVATE_IPV4}:2380"
```

If we give this example to ct with the `--platform=ec2` tag, it produces the following drop-in:

```
[Unit]
Requires=coreos-metadata.service
After=coreos-metadata.service

[Service]
EnvironmentFile=/run/metadata/coreos
Environment="ETCD_IMAGE_TAG=v3.0.15"
ExecStart=
ExecStart=/usr/lib/coreos/etcd-wrapper $ETCD_OPTS \
  --name="${COREOS_EC2_HOSTNAME}" \
  --listen-peer-urls="http://${COREOS_EC2_IPV4_LOCAL}:2380" \
  --listen-client-urls="http://0.0.0.0:2379" \
  --initial-advertise-peer-urls="http://${COREOS_EC2_IPV4_LOCAL}:2380" \
  --initial-cluster="${COREOS_EC2_HOSTNAME}=http://${COREOS_EC2_IPV4_LOCAL}:2380" \
  --advertise-client-urls="http://${COREOS_EC2_IPV4_LOCAL}:2379"
```

This drop-in specifies that etcd should run after the coreos-metadata service, and it uses `/run/metadata/coreos` as an `EnvironmentFile`. This enables the coreos-metadata service, and puts the information it discovers into environment variables. These environment variables are then expanded by systemd when the service starts, inserting the dynamic data into the command-line flags to etcd.

[examples]: #example
