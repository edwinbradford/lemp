# Docker WordPress, NGINX+SSL, MySQL Database.

## Description
Fork of Ron Amosa's dockerized wordpress setup with an NGINX reverse-proxy frontend doing the SSL termination, and a standard MySQL database backend.

There are three major changes in this fork.

1. The original project has no local certificate authority or any instructions on how to create one so instead of certificates and a shell script to generate them please refer to this step by step guide on [Delicious Brains](https://deliciousbrains.com/ssl-certificate-authority-for-local-https-development/) explaining how to generate SSL certificates for local development.
2. The Nginx `default.conf` configuration has deprecated syntax preventing Nginx from running which is resolved in this fork.
3. The Docker Compose wordpress service has a small modification that prevents 'critical errors' being reported in WordPress Site Health tests introduced in WordPress 5.2.

## Pre-requisites
* docker installed locally
* docker-compose installed locally

## Setup
Follow the instructions on [Delicious Brains](https://deliciousbrains.com/ssl-certificate-authority-for-local-https-development/) to create a local certificate authority and certificates, if you choose to use different filenames to those listed below you will need to update the references in Docker Compose and Nginx.

On completing the guide you should end up with the six files listed below. Each file has a `*.example` placeholders in the relevant directories where they should be saved. Please note these `*.example` files are just empty placeholders and won't work, you need to follow the guide.

   * my_wpress_site.crt
   * my_wpress_site.csr
   * my_wpress_site.ext
   * my_wpress_site.key
   * myCA.key
   * myCA.pem

Once you have completed the guide, installed the certificate authorities in your host OS and edited your OS hosts file to include your hostname e.g. `wordpress.localhost` then:

1. Copy the `certs/my_wpress_site.crt` and `certs/my_wpress_site.key` to `nginx/ssl/`, note that the file `my_wpress_site.crt` replaces `my_wpress_site.cert` from the original project.
2. To access WordPress run `docker-compose up --detach` and if one or more of the containers doesn't run look at its logs for errors.

## Hostname
The default hostname `www.mywordpress.local` in the original project is renamed to `wordpress.localhost` in this fork, 'www' is largely deprecated and the top level domain `.local` is not reserved for local development but '.localhost' is.

You will need to add your hostname e.g. `wordpress.localhost` to your system hosts file. In my case that means adding it to the Windows hosts file where my browser runs and the Ubuntu hosts file in [WSL2](https://learn.microsoft.com/en-us/windows/wsl/) where I run Docker.
```
127.0.0.1 wordpress.localhost
```

If all went well you should be able to open a browser at https://wordpress.localhost and you should be greeted (if this is the first time running) with a page asking you to install wordpress.

## Key Points
The following node in the wordpress service of Docker Compose resolves failing WordPress Site Health tests which make loopback requests to localhost inside the Docker wordpress service instead of localhost on the host and therefore through the reverse proxy.
```
extra_hosts:
- wordpress.localhost:172.17.0.1
```

In the Nginx proxy pass directive, 'wordpress' refers to the name of your wordpress service in your docker-compose.yml file:
```
proxy_pass http://wordpress;
```

The following Nginx directives configures WordPress behind a reverse proxy:
```
proxy_set_header      Host $host;
proxy_set_header      X-Real-IP $remote_addr;
proxy_set_header      X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header      X-Forwarded-Host $server_name;
proxy_set_header      X-Forwarded-Proto https;
```

The following conditional in your `wp-config.php` references the Nginx `X-Forwarded-Proto` header above so WordPress can determine SSL requests behind a reverse proxy:
```
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
        $_SERVER['HTTPS'] = 'on';
}
```
