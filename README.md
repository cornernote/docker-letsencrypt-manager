Forked pending this issue: https://github.com/bringnow/docker-letsencrypt-manager/issues/15

# docker-letsencrypt-manager
A docker image allowing you to manage your domains and certificates and automatically renews them. It is based on the [official letsencrypt docker image](https://letsencrypt.readthedocs.org/en/latest/using.html#running-with-docker). It uses the [ACME webroot method](http://letsencrypt.readthedocs.org/en/latest/using.html#webroot) to perform domain validation, allowing zero-downtime certificate renewals.

## Installation

The most simple way to install letsencrypt-manager is by using [docker-compose](https://docs.docker.com/compose/). Letsencrypt-manager already comes with a ready-to-use docker-compose configuration which you can easily adapt to your needs.

### Setting up letsencrypt-manager on a new machine

If you install a machine which should be able to request and automatically renew certificates, you need to first install **letsencrypt-manager** on it. This is easily done by cloning the Git repository and optionally adapting the configuration.

1. Clone letsencrypt-manager
```
# cd /opt/ && git clone https://github.com/cornernote/docker-letsencrypt-manager.git && cd docker-letsencrypt-manager
```
3. Add an alias for letsencrypt-manager (optional):
```
# echo alias letsencrypt-manager=\'/opt/docker-letsencrypt-manager/letsencrypt-manager\' >> ~/.bashrc && source ~/.bashrc
```
4. The command `letsencrypt-manager` is now ready for usage. Type `letsencrypt-manager help` for a list of available commands:
```
root@example.com:~ # letsencrypt-manager help
Checking for newer docker image (pass --no-update-check to suppress this behavior)
Pulling cli (dockersf/letsencrypt-manager:latest)...
latest: Pulling from dockersf/letsencrypt-manager
c30f6751f7b9: Pull complete
e9b49204a716: Pull complete
ffd2bf5cfcb6: Pull complete
5e3aa4c8b310: Pull complete
d03fb0127c24: Pull complete
143361aade13: Pull complete
6aabaf8c992e: Pull complete
bb6cba7c8e81: Pull complete
5135069c23b3: Pull complete
624d6dbcc23e: Pull complete
67c1ab7766ad: Pull complete
f91f147a8823: Pull complete
5716861f8aa4: Pull complete
d518b059a1fd: Pull complete
d8b465d4cc8a: Pull complete
34e75f566970: Pull complete
4e756390d748: Pull complete
1b1ae5ef54d7: Pull complete
Digest: sha256:2692bdd736047fe6028b30f9acc48e774ff03fe1ca7966952450eeb9b4677307
Status: Downloaded newer image for dockersf/letsencrypt-manager:latest
Available commands:

help              - Show this help
list              - List configured domains and their certificate's status
add               - Add a new domain and create a certificate for it
renew             - Renew the certificate for an existing domain. Allows to add additional domain names.
remove            - Remove and existing domain and its certificate
cron-auto-renewal - Run the cron job automatically renewing all certificates
auto-renew        - Try to automatically renew all installed certificates
```

As you can see, the `letsencrypt-manager` helper script automatically downloads the latest version of the docker image. If you want to disable updating on every execution, just pass the `--no-update-check` flag. For example `letsencrypt-manager --no-update-check help`.

You can now proceed with the following sections.

### Custom configuration (optional)

Just define the following environment variables in a file `.env`:

* `LE_EMAIL`: Set the email-address used by letsencrypt. If not set, letsencrypt will ask for it interactively when requesting a certificate. (optional)
* `LE_RSA_KEY_SIZE`: Set the RSA key size used by letsencrypt (optional, defaults to the default setting of letsencrypt).
* `CRON_TIME`: The execution time of the renewal cronjob. For the syntax check [CronHowto](https://help.ubuntu.com/community/CronHowto). Default is weekly, must be entered in standard cron notation * * * * *
* `LE_PRE_HOOK`: Set the hook command for running before a certificate is renewed.
* `LE_POST_HOOK`: Set the hook command for running after a certificate is renewed.
* `AUTO_RENEW_CHECK_SUCCESS_URL`: A URL to send a GET request to after a successful auto renew check (optional). This can be used to get notified if the job fails through a service like healthchecks.io or Dead Man's Snitch.

The `docker-compose.yml` config file already defines some docker host volumes. Of course you can change them easily. See [Compose file reference](https://docs.docker.com/compose/compose-file/#volumes-volume-driver) for syntax details. *Notice that you need to change them twice, for the services cli and cron!*

#### Volumes

* `/etc/letsencrypt`: The configuration directory of the letsencrypt client.
* `/var/lib/letsencrypt`: The working directory of the letsencrypt client.
* `/var/acme-webroot`: This is the directory where letsencrypt puts data for [ACME webroot validation](http://letsencrypt.readthedocs.org/en/latest/using.html#webroot).

### Preparing your webserver

The webserver of your choice must expose the configured webroot folder of the host (see the [Let's Encrypt Documentation](http://letsencrypt.readthedocs.org/en/latest/using.html#webroot) for details).

If you want to use [haproxy](http://www.haproxy.org/) I recommend to use bringnow's [haproxy-letsencrypt Docker image](https://github.com/bringnow/docker-haproxy-letsencrypt) for zero-downtime renewal and automatic reload on configuration/certificate changes.

If you want to use [nginx](http://nginx.org/) I recommend to use bringnow's [nginx-letsencrypt Docker image](https://github.com/bringnow/docker-nginx-letsencrypt) for zero-downtime renewal and automatic reload on configuration/certificate changes.

## Usage

### Show installed domains/certificates

You can show the installed certificates by simply calling `letsencrypt-manager list`:

```
root@example.com:/opt/docker-letsencrypt-manager # letsencrypt-manager list
Checking for newer docker image (pass --no-update-check to suppress this behavior)
Pulling cli (gitsf/letsencrypt-manager:latest)...
latest: Pulling from gitsf/letsencrypt-manager
Digest: sha256:2692bdd736047fe6028b30f9acc48e774ff03fe1ca7966952450eeb9b4677307
Status: Image is up to date for gitsf/letsencrypt-manager:latest
DOMAINNAME                    ALTERNATIVE DOMAINNAMES                        VALID UNTIL               REMAINING DAYS
example.come                  example.com example.de                         Feb 14 12:52:00 2016 GMT  15
```

The helper script will automatically update the docker image of letsencrypt-manager. If you want to skip this update just pass the flag `--no-update-check`.

The `list` command shows a table with one row per certificate and four columns:

1. The **main domain name** of the certificate.
2. The space-separated list of **alternative domain names** for the certificate.
3. The **absolute expiry date** of the certificate.
4. The **relative expiry date** of the certificate in days.

### Adding a new domain/certificate

1. Ensure the DNS configuration for the new domains is properly setup to point to the machine.
2. If you do not have a valid SSL certificate for the domain(s) you want to add,
   just run the ACME enabled webserver without any https configuration (so only port 80).
3. Execute `letsencrypt-manager add <main domain name> [alternative domain names]...`
4. If not previously present, add the https configuration for the added domain(s) to your web server configuration.

### Configuring auto-renewal of certificates

Starting the cronjob which automatically renews all installed certificate is actually quite simple, just execute the `update-cron.sh` script to update the docker image and (re)-creating the service:

```
root@example.com:/opt/docker-letsencrypt-manager # ./update-cron.sh
Checking for newer docker image (pass --no-update-check to suppress this behavior)
latest: Pulling from gitsf/letsencrypt-manager
Digest: sha256:1bca6790d578309fad48ac81c30cae826a91a92d9c09660ca9d307ddc435d6c8
Status: Image is up to date for gitsf/letsencrypt-manager:latest
Creating dockerletsencryptmanager_cron_1...
```

The cronjob will check every day if any of the installed certificates expires in less than 4 weeks and will try to renew them. That was simple, wasn't it?

If you want to manually start the auto-renewal, just call `letsencrypt-manager auto-renew`.

### Modify/renew an existing certificate

To modify the list of alternative domain names and/or manually renew a certificate you can run the `renew` command:

```
letsencrypt-manager renew <domain name> [alternative domain names]...
```

### Removing a domain/certificate

Removing an domain from the host can be achieved by executing `letsencrypt-manager rm <domainname>`. This will remove all certificates and also auto-renewal configuration for this domain.

### Sync certificates, keys and configuration to a Git repository

If you want to backup the private keys and certificates (what you should do!) I recommend bringnow's [docker-git-sync](https://github.com/bringnow/docker-git-sync). It will periodically listen for changes in the */etc/letsencrypt* folder and commit & push any changes to a Git repository of your choice. **Make sure to keep this Git repository in a safe place!**

### Other issue

Please have a look at the [existing issues](https://github.com/gitsf/docker-letsencrypt-manager/issues) or
[report a new one](https://github.com/gitsf/docker-letsencrypt-manager/issues/new).
