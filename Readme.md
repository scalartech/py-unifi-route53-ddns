# `py-unifi-route53-ddns`
This is a minimalistic utility to run dynamic DNS updates on Ubiquiti UniFi Gateway consoles using AWS Route53 DNS.

Ubiquiti UniFi gateways such as [Cloud Gateway Max](https://store.ui.com/us/en/category/cloud-gateways-compact/collections/cloud-gateway-max/products/ucg-max) and [Dream Machine SE](https://store.ui.com/us/en/category/cloud-gateways-large-scale/products/udm-se) provide Internet gateway router functions for home and small business networks. When running the network on an ISP connection without a reserved static IP, you can use dynamic DNS updating to bind the dynamically assigned IP address to a DNS name (such as home.example.net). This DNS name can then be used with a WireGuard configuration to VPN to the network, for example. While the UniFi router software has some built-in connectors to third-party dynamic DNS services, it does not integrate with AWS Route53, which is the DNS provider of choice for many people.

`py-unifi-route53-ddns` uses the system Python to install a virtualenv to isolate its dependencies from the rest of the system, and installs a systemd timer and service (effectively a cron job) to update the DNS hostname every 5 minutes.

It is assumed that you know how to use AWS (including how to create an IAM user and access key, and give your user appropriately scoped permissions to read Route53 hosted zones, read records, and write records - see **IAM permissions** below), and have a hosted zone configured for your domain in Route53.

### Installation
*  Enable SSH in unifi console (navigate to Control Plane -> Console -> Advanced -> SSH), then run `ssh ui@192.168.1.1` and:
```
apt install python3-distutils
python3 -m venv /usr/local/share/pyuir53ddns --without-pip
source /usr/local/share/pyuir53ddns/bin/activate
wget https://bootstrap.pypa.io/get-pip.py
python get-pip.py
pip install https://github.com/cloud-utils/py-unifi-route53-ddns/archive/refs/heads/main.zip
/usr/local/share/pyuir53ddns/bin/py-unifi-route53-ddns install
```
The install script will prompt you for your access key ID, access key, hosted zone domain name, and dynamic hostname to update. These variables will be saved to the systemd service override file in `/etc/systemd/system/py-unifi-route53-ddns.service.d/env.conf`. Other files created by the service are:

* `/etc/systemd/system/py-unifi-route53-ddns.service`
* `/etc/systemd/system/py-unifi-route53-ddns.timer`
* `/usr/local/share/pyuir53ddns`, the virtualenv, as seen above

To remove the service, just delete all of these files.

### Monitoring
Use `systemctl status py-unifi-route53-ddns.service` or `journalctl -u py-unifi-route53-ddns.service` to see the status and logs of the service.

### WireGuard VPN configuration
The UniFi console provides a built-in WireGuard VPN. Navigate to Control Plane -> VPN -> VPN Server -> Create New, configure the server, and check "Use Alternate Address for Clients", then enter the FQDN that you configured as the dynamic hostname above. Any client added after this point (with a QR code or otherwise) will receive this configuration.

### IAM permissions
Use the visual editor to create a policy with the following permissions:
* Route53 `ListHostedZonesByName`
* Route53 `ListResourceRecordSets`
* Route53 `ChangeResourceRecordSets`

When asked for the resource, specify the zone ID of the Route53 hosted zone that you're using.

Or use the following policy JSON:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets",
                "route53:ListResourceRecordSets"
            ],
            "Resource": "arn:aws:route53:::hostedzone/REPLACE_WITH_YOUR_HOSTED_ZONE_ID"
        },
        {
            "Effect": "Allow",
            "Action": "route53:ListHostedZonesByName",
            "Resource": "*"
        }
    ]
}
```

### Bugs

Please report bugs, issues, feature requests, etc. on [GitHub](https://github.com/cloud-utils/py-unifi-route53-ddns/issues).

### License

Copyright 2024, Andrey Kislyuk and py-unifi-route53-ddns contributors. Licensed under the terms of the
[Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0). Distribution of the LICENSE and NOTICE
files with source copies of this package and derivative works is **REQUIRED** as specified by the Apache License.
