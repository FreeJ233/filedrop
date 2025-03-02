# filedrop

Easy peer-to-peer file transfer.

<p align="center">
    <a href="https://drop.lol/">
        <strong>Click here to open drop.lol.</strong>
    </a>
</p>

<p align="center">
    <a href="https://drop.lol/">
        <img src="https://raw.githubusercontent.com/mat-sz/filedrop/master/filedrop.gif" alt="Screenshot">
    </a>
</p>

## Self-hosting

### Docker

#### Requirements:

- docker
- docker-compose
- bash
- openssl

Clone the repo and run the following command:

```
./docker-start.sh
```

Make sure that your user is in the docker group.

In case another reverse proxy is used make sure to change the default port (from 80) and to add the `X-Forwarded-For` header with client's IP address.

TURN uses TCP port 3478 and UDP ports 49152-65535.

#### docker-start.sh arguments

| Option       | Description                                     |
| ------------ | ----------------------------------------------- |
| `-n <name>`  | Sets application name.                          |
| `-e <email>` | Sets contact email.                             |
| `-p <port>`  | Sets port for the application to be exposed at. |
| `-f`         | Enables WS_USE_X_FORWARDED_FOR.                 |
| `-s`         | Enables WS_REQUIRE_CRYPTO.                      |

### Manual

> First you need to set up a TURN server (like [coturn](https://github.com/coturn/coturn)).
>
> Then you need to clone this repository, run `yarn build` and then `yarn start`. I also use nginx to proxy the back end through it. [Here's a guide on how to achieve that.](https://www.nginx.com/blog/websocket-nginx/)

### Environment variables

The following variables are used in the build process of the frontend:

| Variable        | Default value | Description                                   |
| --------------- | ------------- | --------------------------------------------- |
| `VITE_APP_NAME` | `filedrop`    | Default application title (while connecting). |

The following variables are used in the WebSockets server:

| Variable                 | Default value                   | Description                                                                                                               |
| ------------------------ | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `WS_APP_NAME`            | `filedrop`                      | Application title.                                                                                                        |
| `WS_ABUSE_EMAIL`         | null                            | E-mail to show in the Abuse section.                                                                                      |
| `WS_HOST`                | `127.0.0.1`                     | IP address to bind to.                                                                                                    |
| `WS_PORT`                | `5000`                          | Port to bind to.                                                                                                          |
| `WS_USE_X_FORWARDED_FOR` | `0`                             | Set to `1` if you want the application to respect the `X-Forwarded-For` header.                                           |
| `WS_MAX_SIZE`            | `65536`                         | The limit should accommodate preview images (100x100 thumbnails).                                                         |
| `WS_MAX_NETWORK_CLIENTS` | `64`                            | Limits the amount of clients that can connect to one room.                                                                |
| `WS_REQUIRE_CRYPTO`      | `0`                             | Set to `1` if you want to ensure that all communication between clients is encrypted. HTTPS is required for this to work. |
| `STUN_SERVER`            | `stun:stun1.l.google.com:19302` | STUN server address.                                                                                                      |
| `TURN_MODE`              | `default`                       | `default` for static credentials, `hmac` for time-limited credentials.                                                    |
| `TURN_SERVER`            | null                            | TURN server address.                                                                                                      |
| `TURN_USERNAME`          | null                            | TURN username.                                                                                                            |
| `TURN_CREDENTIAL`        | null                            | TURN credential (password).                                                                                               |
| `TURN_SECRET`            | null                            | TURN secret (required for `hmac`).                                                                                        |
| `TURN_EXPIRY`            | `3600`                          | TURN token expiration time (when in `hmac` mode), in seconds.                                                             |
| `NOTICE_TEXT`            | null                            | Text of the notice to be displayed for all clients.                                                                       |
| `NOTICE_URL`             | null                            | URL the notice should link to.                                                                                            |

## FAQ

### What is the motivation behind the project?

I didn't feel comfortable logging into my e-mail account on devices I don't own just to download an attachment and cloud services have extremely long URLs that aren't really easy to type.

### Where do my files go after I send them through the service?

To the other device. Sometimes the (encrypted, since WebRTC uses encryption by default) data goes through the TURN server I run. It's immediately discarded after being relayed. File metadata also is not saved.

### Doesn't this exist already?

While [ShareDrop](https://github.com/cowbell/sharedrop) and [SnapDrop](https://github.com/RobinLinus/snapdrop) are both excellent projects and most definitely exist, I felt the need to create my own version for a several reasons:

- I wanted to build something using React.js and TypeScript.
- ShareDrop doesn't work when the devices are on different networks but still behind NAT.
- I didn't like the layout and design of both, I feel like the abstract design of FileDrop makes it easier to use.
- I was not aware of these projects while I started working on this project.
- ShareDrop's URLs are extremely long.

### How is it related to the other projects you've mentioned?

I don't use PeerJS (while the other two projects do) and I also host TURN and WebSocket servers myself (instead of relying on Firebase). Sometimes you may get connected to Google's STUN server (always if a TURN server is not provided in the configuration).

## HTTPS setup

### Setup with a reverse proxy in front of nginx

1. Configure your reverse proxy to proxy requests to `127.0.0.1:PORT` and then follow your usual instructions for using SSL certificates with said proxy.
2. Rebuild the application.
3. Make sure the TURN server can be connected to from the outside.

#### Nginx configuration example

More details available here: https://www.nginx.com/blog/websocket-nginx/

```nginx
worker_processes auto;

events {
  worker_connections 1024;
}

http {
  upstream filedrop {
    server 127.0.0.1:5000; # 5000 = PORT
  }

  map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
  }

  # ...

  server {
    listen 80;
    # server_name should be configured here.
    # HTTPS should be configured here. (certbot will handle this for you, if you're using Let's Encrypt.)

    # ...

    location / {
      proxy_pass http://filedrop;
      proxy_http_version 1.1;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Host $host;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection $connection_upgrade;
    }
  }
}
```
