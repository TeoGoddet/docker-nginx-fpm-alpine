# PrivateBin on nginx, php-fpm & alpine

**PrivateBin** is a minimalist, open source online [pastebin](https://en.wikipedia.org/wiki/Pastebin) where the server has zero knowledge of pasted data. Data is encrypted and decrypted in the browser using 256bit AES in [Galois Counter mode](https://en.wikipedia.org/wiki/Galois/Counter_Mode).

This repository contains the Dockerfile and resources needed to create a docker image with a pre-installed PrivateBin instance in a secure default configuration. The images are based on the docker hub alpine image, extended with the GD module required to generate discussion avatars and the Nginx webserver to serve static JavaScript libraries, CSS & the logos. All logs of php-fpm and Nginx (access & errors) are forwarded to docker logs.

## Running the image

Assuming you have docker successfully installed and internet access, you can fetch and run the image from the docker hub like this:

```bash
docker run -d --restart="always" --read-only -p 8080:80 -v privatebin-data:/srv/data privatebin/nginx-fpm-alpine
```

The parameters in detail:

- `-v privatebin-data:/srv/data` - replace `privatebin-data` with the path to the folder on your system, where the pastes and other service data should be persisted. This guarantees that your pastes aren't lost after you stop and restart the image or when you replace it. May be skipped if you just want to test the image.
- `-p 8080:80` - The Nginx webserver inside the container listens on port 80, this parameter exposes it on your system on port 8080. Be sure to use a reverse proxy for HTTPS termination in front of it for production environments.
- `--read-only` - This image supports running in read-only mode. Using this reduces the attack surface slightly, since an exploit in one of the images services can't overwrite arbitrary files in the container. Only /tmp, /var/tmp, /var/run & /srv/data may be written into.
- `-d` - launches the container in the background. You can use `docker ps` and `docker logs` to check if the container is alive and well.
- `--restart="always"` - restart the container if it crashes, mainly useful for production setups

> Note that the volume mounted must be owned by UID 65534 / GID 82. If you run the container in a docker instance with "userns-remap" you need to add your subuid/subgid range to these numbers.

### Custom configuration

In case you want to use a customized [conf.php](https://github.com/PrivateBin/PrivateBin/blob/master/cfg/conf.sample.php) file, for example one that has file uploads enabled or that uses a different template, add the file as a second volume:

```bash
docker run -d --restart="always" --read-only -p 8080:80 -v conf.php:/srv/cfg/conf.php:ro -v privatebin-data:/srv/data privatebin/nginx-fpm-alpine
```

Note: The `Filesystem` data storage is supported out of the box. The image includes PDO modules for MySQL, PostgreSQL and SQLite, required for the `Database` one, but you still need to keep the /srv/data persisted for the server salt and the traffic limiter.

### Adjusting nginx or php-fpm settings

You can attach your own `php.ini` or nginx configuration files to the folders `/etc/php7/conf.d/` and `/etc/nginx/conf.d/` respectively. This would for example let you adjust the maximum size these two services accept for file uploads, if you need more then the default 10 MiB.

### Timezone settings

The image supports the use of the following two environment variables to adjust the timezone. This is most useful to ensure the logs show the correct local time.

- `TZ`
- `PHP_TZ`

Note: The application internally handles expiration of pastes based on a UNIX timestamp that is calculated based on the timezone set during its creation. Changing the PHP_TZ will affect this and leads to earlier (if the timezone is increased) or later (if it is decreased) expiration then expected.

## Rolling your own image

To reproduce the image, run:

```bash
docker build -t privatebin/nginx-fpm-alpine .
```

### Behind the scenes

The two processes, Nginx and php-fpm, are started by supervisord, which will also try to restart the services in case they crash.

Nginx is required to serve static files and caches them, too. Requests to the index.php (which is the only PHP file exposed in the document root at /var/www) are passed on to php-fpm via fastCGI to port 9000. All other PHP files and the data are stored in /srv.

The Nginx setup supports only HTTP, so make sure that you run another webserver as reverse proxy in front of this for HTTPS offloading and reducing the attack surface on your TLS stack. The Nginx in this image is set up to deflate/gzip text content.

During the build of the image, the opcache & GD PHP modules are compiled from source and the PrivateBin release archive is downloaded from Github. All the downloaded Alpine packages and the PrivateBin archive are validated using cryptographic signatures to ensure the have not been tempered with, before deploying them in the image.
