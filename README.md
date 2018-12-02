# Summary

This container is designed to run along side your Nextcloud container to execute its
`/var/www/html/cron.php` at a regular interval. There is an "official" way of doing this, however it
doesn't work when you run your Nextcloud container using a non-root user. I also personally feel
that this solution is easier to manage, since it doesn't require the same environment as Nextcloud
itself (i.e. no network requirements, no database requirements, etc).

# Setup Instructions

Since Nextcloud's entire setup can get rather complex with Docker, I highly recommend you set up
everything using [Docker Compose](https://docs.docker.com/compose/).

Below is an example of how you set up your `docker-compose.yml` to work with Nextcloud using this
container. Note that the `app` service is greatly simplified for example purposes. It is only to
show usage of the cronjob image in conjunction with your Nextcloud container.

```yml
version: '3.7'

services:
  app:
    image: nextcloud:apache

  cron:
    image: voidpointer/nextcloud-cronjob
    restart: always
    network_mode: none
    depends_on:
      - app
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /etc/localtime:/etc/localtime:ro
    environment:
      - NEXTCLOUD_CONTAINER_NAME=app
      - NEXTCLOUD_PROJECT_NAME=nextcloud
```

In this example, the `cron` service runs with a dependency on `app` (which is Nextcloud itself).
Every 15 minutes (default) the `cron` service will execute `php -f /var/www/html/cron.php` via the
`docker exec` command. The `NEXTCLOUD_CONTAINER_NAME` and `NEXTCLOUD_PROJECT_NAME` work together to
help identify the right container to execute the command in.

Note that if you don't use Docker Compose, you can leave `NEXTCLOUD_PROJECT_NAME` blank or omitted
entirely.

# Environment Variables

* `NEXTCLOUD_CONTAINER_NAME`<br>
  Required. This is the name of the running Nextcloud container (or
  the service, if `NEXTCLOUD_PROJECT_NAME` is specified).
* `NEXTCLOUD_PROJECT_NAME`<br>
  The name of the project if you're using Docker Compose. The name of
  the project, by default, is the name of the context directory you ran your `docker-compose.yml`
  from. This helps to build a "hint" used to identify the Nextcloud container by name. The hint is
  built as:

      ${NEXTCLOUD_PROJECT_NAME}_${NEXTCLOUD_CONTAINER_NAME}
* `NEXTCLOUD_CRON_MINUTE_INTERVAL`<br>
  The interval, in minutes, of how often the cron task
  executes. The default is 15 minutes.

# Container Health

If you do `docker-compose ps`, you will see the active health of the container. The following logic
is checked every interval of the health check. If any of these checks fail, it is likely the
container's health status will become *unhealthy*. In this case, you should restart the container.

1. The `crond` process must be running.
2. The Nextcloud container must be available and running. One important note here: When this
   container starts up, it immediately searches for the container by name and remembers it by the
   container's ID. If for whatever reason the Nextcloud container changes in such a way that the ID
   is no longer valid, the health check would fail.

# Debugging

All logs from `crond` are configured to print to stdout, so you can monitor container logs (via
`docker-compose logs -f`). This should allow you to make sure your cron job is working. You can also
use the "Overview" page in Nextcloud Settings to see if the cron job is being run regularly.