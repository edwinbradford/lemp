# Docker WordPress, NGINX+SSL, MySQL Database.

## Description
Fork of Ron Amosa's dockerized wordpress setup with an NGINX reverse-proxy frontend doing the SSL termination, and a standard MySQL database backend.

There are three major changes from the original project.

1. The original project has no certificate authority or instructions to create one without which the cerfificates can't be used so they are deleted from the project and replaced by a link to step by step instructions explaining how to generate SSL certificates for local development.
2. The Nginx `default.conf` configuration had deprecated syntax preventing Nginx from running which is removed.
3. The Docker Compose wordpress service has a simple modification that enables support for WordPress Site Health tests introduced in WordPress 5.2.

## Pre-requisites
* docker installed locally
* docker-compose installed locally

## Setup
Follow the instructions from the website linked to in the `README.md` in the `certs` directory to create a local certificate authority and certificates, if you choose to use different filenames to those below you will need to update the references in Docker Compose and Nginx. You should end up with six files, none of which should really be stored in GitHub:
   * my_wpress_site.crt
   * my_wpress_site.csr
   * my_wpress_site.ext
   * my_wpress_site.key
   * myCA.key
   * myCA.pem

1. Copy the `certs/my_wpress_site.crt` and `certs/my_wpress_site.key` to `nginx/ssl/`, the file `my_wpress_site.crt` replaces `my_wpress_site.cert` from the original project.
2. Run `docker-compose up --detach` and if one or more of the containers doesn't run look at its logs for the error.

## Setup Wordpress
The default hostname `www.mywordpress.local` from the original project is changed to `wordpress.localhost`, 'www' is largely deprecated now and the top level domain `.local` is not reserved for local development whilst '.localhost' is.

You will need to add your hostname e.g. `wordpress.localhost` to your system hosts file. In my case that means adding it to Windows where my browser runs and Ubuntu in [WSL2](https://learn.microsoft.com/en-us/windows/wsl/) where I run Docker.
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
