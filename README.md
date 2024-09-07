# L2TP server for routers without IPSec / L2TP сервер для роутеров без IPSec 
# by boumer7

[Оригинальный репозиторий](https://github.com/hwdsl2/docker-ipsec-vpn-server)
и
[Issue](https://github.com/hwdsl2/docker-ipsec-vpn-server/issues/191)

## Quick start

Use this command to set up an IPsec VPN server on Docker:

```
docker run \
    --name ipsec-vpn-server \
    --restart=always \
    -v ikev2-vpn-data:/etc/ipsec.d \
    -v /lib/modules:/lib/modules:ro \
    -p 500:500/udp \
    -p 4500:4500/udp \
    -d --privileged \
    hwdsl2/ipsec-vpn-server
```

Your VPN login details will be randomly generated. See [Retrieve VPN login details](#retrieve-vpn-login-details).

Alternatively, you may [set up IPsec VPN without Docker](https://github.com/hwdsl2/setup-ipsec-vpn). To learn more about how to use this image, read the sections below.

## Features

- Supports IKEv2 with strong and fast ciphers (e.g. AES-GCM)
- Generates VPN profiles to auto-configure iOS, macOS and Android devices
- Supports Windows, macOS, iOS, Android, Chrome OS and Linux as VPN clients
- Includes a helper script to manage IKEv2 users and certificates

## Install Docker

First, [install Docker](https://docs.docker.com/engine/install/) on your Linux server. You may also use [Podman](https://podman.io) to run this image, after [creating an alias](https://podman.io/whatis.html) for `docker`.

Advanced users can use this image on macOS with [Docker for Mac](https://docs.docker.com/docker-for-mac/). Before using IPsec/L2TP mode, you may need to restart the Docker container once with `docker restart ipsec-vpn-server`. This image does not support Docker for Windows.

## Download

Get the trusted build from the [Docker Hub registry](https://hub.docker.com/r/hwdsl2/ipsec-vpn-server/):

```
docker pull hwdsl2/ipsec-vpn-server
```

Alternatively, you may download from [Quay.io](https://quay.io/repository/hwdsl2/ipsec-vpn-server):

```
docker pull quay.io/hwdsl2/ipsec-vpn-server
docker image tag quay.io/hwdsl2/ipsec-vpn-server hwdsl2/ipsec-vpn-server
```

Supported platforms: `linux/amd64`, `linux/arm64` and `linux/arm/v7`.

Advanced users can [build from source code](docs/advanced-usage.md#build-from-source-code) on GitHub.

### Image comparison

Two pre-built images are available. The default Alpine-based image is only ~18 MB.

|                   | Alpine-based             | Debian-based                   |
| ----------------- | ------------------------ | ------------------------------ |
| Image name        | hwdsl2/ipsec-vpn-server  | hwdsl2/ipsec-vpn-server:debian |
| Compressed size   | ~ 18 MB                  | ~ 63 MB                        |
| Base image        | Alpine Linux 3.20        | Debian Linux 12                |
| Platforms         | amd64, arm64, arm/v7     | amd64, arm64, arm/v7           |
| Libreswan version | 5.0                      | 5.0                            |
| IPsec/L2TP        | ✅                       | ✅                              |
| Cisco IPsec       | ✅                       | ✅                              |
| IKEv2             | ✅                       | ✅                              |

**Note:** To use the Debian-based image, replace every `hwdsl2/ipsec-vpn-server` with `hwdsl2/ipsec-vpn-server:debian` in this README. These images are not currently compatible with Synology NAS systems.

<details>
<summary>
I want to use the older Libreswan version 4.
</summary>

It is generally recommended to use the latest [Libreswan](https://libreswan.org/) version 5, which is the default version in this project. However, if you want to use the older Libreswan version 4, you can build the Docker image from source code:

```
git clone https://github.com/hwdsl2/docker-ipsec-vpn-server
cd docker-ipsec-vpn-server
# Specify Libreswan version 4
sed -i 's/SWAN_VER 5\.0/SWAN_VER 4.15/' Dockerfile Dockerfile.debian
# To build Alpine-based image
docker build -t hwdsl2/ipsec-vpn-server .
# To build Debian-based image
docker build -f Dockerfile.debian -t hwdsl2/ipsec-vpn-server:debian .
```
</details>

## How to use this image

### Environment variables

**Note:** All the variables to this image are optional, which means you don't have to type in any variable, and you can have an IPsec VPN server out of the box! To do that, create an empty `env` file using `touch vpn.env`, and skip to the next section.

This Docker image uses the following variables, that can be declared in an `env` file (see [example](vpn.env.example)):

```
VPN_IPSEC_PSK=your_ipsec_pre_shared_key
VPN_USER=your_vpn_username
VPN_PASSWORD=your_vpn_password
```

This will create a user account for VPN login, which can be used by your multiple devices[*](#important-notes). The IPsec PSK (pre-shared key) is specified by the `VPN_IPSEC_PSK` environment variable. The VPN username is defined in `VPN_USER`, and VPN password is specified by `VPN_PASSWORD`.

Additional VPN users are supported, and can be optionally declared in your `env` file like this. Usernames and passwords must be separated by spaces, and usernames cannot contain duplicates. All VPN users will share the same IPsec PSK.

```
VPN_ADDL_USERS=additional_username_1 additional_username_2
VPN_ADDL_PASSWORDS=additional_password_1 additional_password_2
```

**Note:** In your `env` file, DO NOT put `""` or `''` around values, or add space around `=`. DO NOT use these special characters within values: `\ " '`. A secure IPsec PSK should consist of at least 20 random characters.

**Note:** If you modify the `env` file after the Docker container is already created, you must remove and re-create the container for the changes to take effect. Refer to [Update Docker image](#update-docker-image).

### Additional environment variables

Advanced users can optionally specify a DNS name, client name and/or custom DNS servers.

<details>
<summary>
Learn how to specify a DNS name, client name and/or custom DNS servers.
</summary>

Advanced users can optionally specify a DNS name for the IKEv2 server address. The DNS name must be a fully qualified domain name (FQDN). Example:

```
VPN_DNS_NAME=vpn.example.com
```

You may specify a name for the first IKEv2 client. Use one word only, no special characters except `-` and `_`. The default is `vpnclient` if not specified.

```
VPN_CLIENT_NAME=your_client_name
```

By default, clients are set to use [Google Public DNS](https://developers.google.com/speed/public-dns/) when the VPN is active. You may specify custom DNS server(s) for all VPN modes. Example:

```
VPN_DNS_SRV1=1.1.1.1
VPN_DNS_SRV2=1.0.0.1
```

For more details and a list of some popular public DNS providers, see [Use alternative DNS servers](docs/advanced-usage.md).

By default, no password is required when importing IKEv2 client configuration. You can choose to protect client config files using a random password.

```
VPN_PROTECT_CONFIG=yes
```

**Note:** The variables above have no effect for IKEv2 mode, if IKEv2 is already set up in the Docker container. In this case, you may remove IKEv2 and set it up again using custom options. Refer to [Configure and use IKEv2 VPN](#configure-and-use-ikev2-vpn).
</details>

### Start the IPsec VPN server

Create a new Docker container from this image (replace `./vpn.env` with your own `env` file):

```
docker run \
    --name ipsec-vpn-server \
    --env-file ./vpn.env \
    --restart=always \
    -v ikev2-vpn-data:/etc/ipsec.d \
    -v /lib/modules:/lib/modules:ro \
    -p 500:500/udp \
    -p 4500:4500/udp \
    -d --privileged \
    hwdsl2/ipsec-vpn-server
```

In this command, we use the `-v` option of `docker run` to create a new [Docker volume](https://docs.docker.com/storage/volumes/) named `ikev2-vpn-data`, and mount it into `/etc/ipsec.d` in the container. IKEv2 related data such as certificates and keys will persist in the volume, and later when you need to re-create the Docker container, just specify the same volume again.

It is recommended to enable IKEv2 when using this image. However, if you prefer not to enable IKEv2 and use only the IPsec/L2TP and IPsec/XAuth ("Cisco IPsec") modes to connect to the VPN, remove the first `-v` option from the `docker run` command above.

**Note:** Advanced users can also [run without privileged mode](docs/advanced-usage.md#run-without-privileged-mode).

### Retrieve VPN login details

If you did not specify an `env` file in the `docker run` command above, `VPN_USER` will default to `vpnuser` and both `VPN_IPSEC_PSK` and `VPN_PASSWORD` will be randomly generated. To retrieve them, view the container logs:

```
docker logs ipsec-vpn-server
```

Search for these lines in the output:

```
Connect to your new VPN with these details:

Server IP: your_vpn_server_ip
IPsec PSK: your_ipsec_pre_shared_key
Username: your_vpn_username
Password: your_vpn_password
```

The output will also include details for IKEv2 mode, if enabled.

(Optional) Backup the generated VPN login details (if any) to the current directory:

```
docker cp ipsec-vpn-server:/etc/ipsec.d/vpn-gen.env ./
```

## Technical details / Тех. детали

There are two services running: `Libreswan (pluto)` for the IPsec VPN, and `xl2tpd` for L2TP support.

The default IPsec configuration supports:

* IPsec/L2TP with PSK
* IKEv1 with PSK and XAuth ("Cisco IPsec")
* IKEv2

The ports that are exposed for this container to work are:

порты 1701:1701/udp -p 500:500/udp -p 4500:4500/udp

## License / Лицензия

**Note:** The software components inside the pre-built image (such as Libreswan and xl2tpd) are under the respective licenses chosen by their respective copyright holders. As for any pre-built image usage, it is the image user's responsibility to ensure that any use of this image complies with any relevant licenses for all software contained within.

Copyright (C) 2016-2024 [Lin Song](https://github.com/hwdsl2) [![View my profile on LinkedIn](https://static.licdn.com/scds/common/u/img/webpromo/btn_viewmy_160x25.png)](https://www.linkedin.com/in/linsongui)   
Based on [the work of Thomas Sarlandie](https://github.com/sarfata/voodooprivacy) (Copyright 2012)

[![Creative Commons License](https://i.creativecommons.org/l/by-sa/3.0/88x31.png)](http://creativecommons.org/licenses/by-sa/3.0/)   
This work is licensed under the [Creative Commons Attribution-ShareAlike 3.0 Unported License](http://creativecommons.org/licenses/by-sa/3.0/)   
Attribution required: please include my name in any derivative and let me know how you have improved it!
