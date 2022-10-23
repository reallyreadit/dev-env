# Readup Development Environment Setup Guide

## Core repositories

Readup will run on Linux, macOS or Windows. There are three main repositories that constitute the core of the Readup platform: [`db`](https://github.com/reallyreadit/db), [`api`](https://github.com/reallyreadit/api) and [`web`](https://github.com/reallyreadit/web).

Begin setting up the development environment by cloning these three repositories, and some other ones that will be needed for the Docker setup, preferably into the same parent directory:

```
mkdir readup
cd readup
for repo in db api web dev-env static blog; do git clone https://github.com/reallyreadit/$repo; done
```

Next, you have a choice on how to set up the development environment.

The easiest way is to use [Docker](https://www.docker.com/get-started/) and our [Docker Compose](https://docs.docker.com/compose/gettingstarted/) configuration. With one command, this will install and run the required server containers to get started with development: the node.js 14 `web` server, ASP.NET 3.1 `api` server, Postgres 14 `db` server, and a reverse proxy server that provides a HTTPS interface to the previous services. See "Install via Docker" below.

Alternatively, you can manually install all these dependencies by following the readmes of [`db`](https://github.com/reallyreadit/db), [`api`](https://github.com/reallyreadit/api) and [`web`](https://github.com/reallyreadit/web), in that order. Then skip to "Manual Installation".

## Quickstart: install via Docker

### 1. Run the containers with Docker Compose

#### Configuring the `api`

Run these commands in the the root of this `dev-env`. Make sure that the `web`, `dev` and `api` repository folders are siblings of the `dev-env` folder, otherwise the relative folder references in the Compose file will not work. If you don't use a UNIX-style CLI (Windows), change the commands as appropriate:

Enable the default config for Docker in `api`
```
cd ../api
cp ./appsettings.docker.json appsettings.json
cp ./hostsettings.docker.json hostsettings.json
cd -
```

#### Configuring the `web` app

The default `web` docker-compose setup maps the `../web` host directory onto the docker runtime web app root, to enable live-server development from the start. As a consequence, if you never ran `npm ci` or `npm install` in `web` on your host machine, the docker runtime won't have installed dependencies.

The surest way to proceed is to open `web` as a VSCode dev container, and run `npm ci` within the container. The container has the needed node/npm runtime preinstalled and will write the `node_modules` to your host filesystem.

If you have node v14 installed on your host, you cal also just run `npm ci` in `web` without using container.
#### Starting all services

Set up all services with Docker Compose:

```
docker-compose -p readup -f docker-compose.yml up --build -d
```
(or `docker compose` without the dash, in case you have the modern plugin installed)

This command will download, install, configure and run the node.js 14 `web` server, ASP.NET `api` server, Postgres 14 `db` server, and a reverse proxy server.

_Note_: if you get an error on macOS Monterey (or later) that port 5000 is already in use; this is caused by the AirPlay Receiver. You can disable the receiver temporarily in the System Preferences - Sharing, then start the API container again. After this, re-enable the receiver, it will pick a different port.

With the default Dockerfiles, `web` will run a development server that can be used for basic web app development purposes. To develop the Readup embed, web extension, desktop or iOS app, you need to read the docs of the `web` and other relevant repositories (`ios`, `desktop`) to learn how to further configure your environment, and which build commands to run inside the container (some Docker experience is helpful).

### 2. Trust Readup's SSL root certificate authority

You could technically run Readup at this point, but the default web server configuration makes Readup run via `https` (and not `http`), because browsers treat websites served over an insecure connection with special consideration. To enable HTTPS for development purposes, Readup uses a self-signed certificate included in this `dev-env` repository.

In order to make sure that your system accepts this self-signed ceritficate, `ca.dev.reallyread.it.cer` from the `ssl` directory in this repo should be added to your system as a trusted root certificate authority. Here's how:

- macOS
	- Import into "System" in Keychain Access
	- Right-click certificate and select "Get Info"
	- Expand the "Trust" section and select "Always Trust"
- Windows: import into the local computer's "Trusted Root Certification Authorities" location.
- Linux: in the case of a Ubuntu-based distro, check [these official instructions](https://ubuntu.com/server/docs/security-trust-store). If you're using another distro, you can probably find the answer with a web search yourself! This worked on ElementaryOS Jolnir with ca-certificates pre-installed:
   ```
   sudo cp ca.dev.reallyread.it.cer /usr/local/share/ca-certificates/ca.dev.reallyread.it.crt
   sudo update-ca-certificates
   ```

Most browsers will accept the certificate once added to the system certificate store.

Firefox on macOS uses its own certificate store and requires the following steps.
1. Open "Preferences" and go to "Privacy & Security"
2. Scroll down to the "Certificates" section and click "View Certificates..."
3. Go to the "Authorities" tab and click "Import" to import the `ca.dev.reallyread.it.cer` certificate.

Similary, Chrome on Ubuntu also seems to use its own store (at least with ElementaryOS 6.1).
1. In the settings, go to "Privacy & Security"
2. Open "Security", then scroll down to open "Manage Certificates"
3. Click the tab "Authorities"
4. Import `ca.dev.reallyread.it.cer`. You may need to enable "All Files" in your file manager, as the `.cer` extension could be unexpected by Chrome.

Other browsers on other operating systems may need similar steps.

### 3. Set up your hosts file

Again, to better match the production environment, the default development configuration uses the `dev.readup.com` domain name (with `https`) rather than `localhost` because websites running on `localhost` and served over non-default ports are treated with special care.

Add the following entries to your hosts file located at `/private/etc/hosts` (macOS), `/etc/hosts` (most Linux distributions) or `C:\Windows\System32\drivers\etc\hosts` (Windows):

```
127.0.0.1 api.dev.readup.com
127.0.0.1 dev.readup.com
127.0.0.1 static.dev.readup.com
127.0.0.1 blog.dev.readup.com
127.0.0.1 article-test.dev.readup.com
127.0.0.1 twitter-test.dev.readup.com
```
### 4. Done!?

You should now be able to navigate to [https://dev.readup.com](https://dev.readup.com) in your browser, and see the Readup homepage.

Running [https://dev.readup.com/?clientType=App](https://dev.readup.com/?clientType=App) should allow you to create a new Readup account in your development environment and open the Readup web app. You can also log into the default testing user of the seed data with the username `sample@email.com` and password `password`.

Let us know [on our Discord](https://discord.gg/GJkaPfdSFE) if there were any issues!

## Other Docker services

### Blog

The Readup blog, served at [https://blog.readup.com](https://blog.readup.com), is a Jekyll site.

The blog code lives in this repository: [https://github.com/reallyreadit/blog](https://github.com/reallyreadit/blog).

A development container is not started by default, but can be manually started with:

```
docker compose -p readup up blog -d
```

The default reverse-proxy server is set to point [https://blog.dev.readup.com](https://blog.dev.readup.com) to the build output directory. Useful for testing the embed.

### Mail server

The `api` server might send emails on certain events, for example, on the creation of an account. To intercept and debug these emails, the Compose project also includes an optional [mailcatcher](https://mailcatcher.me/) container. The default Docker API server config is already configured to connect to it.

Start this container manually with:

```
docker compose -p readup up mail -d
```

Then view intercepted emails at [http://localhost:1080](http:://localhost:1080)

### Article test server

Configure:

```
cd ../article-test-server
cp ./appsettings.docker.json appsettings.json
cd -
```

Then start it manually with:

```
docker compose -p readup up article-test-server -d
```

## Manual Installation

Refer to this section when you can not (or don't want to) use Docker, or when you want to understand the inner workings of the Readup dev env for more advanced configuration.

This section assumes that you have already worked through the following the readmes of [`db`](https://github.com/reallyreadit/db), [`api`](https://github.com/reallyreadit/api) and [`web`](https://github.com/reallyreadit/web), in that order.

The guides in the readme files use macOS commands and directory structures as examples so translations may be required for Linux and Windows systems. PowerShell is required to execute some scripts that are included in various repositories. It is included by default in Windows but will need to be installed separately on Linux and macOS systems: https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell

Once you completed manual set-up instructions of readmes, you'll have database, API and web servers running locally. For the reasons outlined in the Docker installation steps 4 & 5, you'll need to manually set up a reverse proxy that allows us to host the Readup web app at `https://dev.readup.com`, locally.

First, complete those steps (trust the SSL certificate and set up the hosts file). Then refere

### Static Content
The `static.readup.com` production server serves four main purposes:

1. **Hosting font and image files used by the various Readup clients.**

	 These files are included in this repository in the `static` directory. This will be the root directory of our `static.readup.com` development server.
2. **Hosting the authentication service popup handler HTML file.**

    This is a single static source file in the `web` repository that is not currently handled by the build system. We'll set up an alias at `/common/auth-service-popup-handler/v1/index.html` that points to the source file.
3. **Hosting the embed script files.**

    These files are generated by the `web` repository build system. We'll set up a virtual directory at `/embed` that points to the build output directory.
4. **Hosting extension and native client script files and indexes of those files in order to facilitate over-the-air updates.**

    Setup of these directories is currently a manual process not covered in this guide. Not required for development use except for testing of the script update functionality.

For development purposes, the static content is served from the `content` directory in the `static`, by the reverse nginx proxy detailed next.

### Reverse-Proxy Web Server

This setup guide uses nginx but any web server capable of acting as a reverse-proxy and serving static content will work. The directories and commands are for macOS and will need to be modified for other operating systems.
1. Install nginx using Macports or Homebrew. Macports installation is as follows:

    1. Install Macports: https://www.macports.org/install.php The installer should add `/opt/local/bin` and `/opt/local/sbin` to your `PATH`.
	 2. Install nginx: `sudo port install nginx`
	 3. Start nginx: `sudo port load nginx`
	 4. Verify nginx is installed and running by visiting http://127.0.0.1
2. Copy the `dev.readup.com.cer` and `dev.readup.com.key` files in the `ssl` directory to `/etc/ssl`
3. Create a custom nginx configuration file at `/opt/local/etc/nginx/readup-server.conf` with the following content:
    ```
    listen              443 ssl;
    ssl_certificate     /etc/ssl/dev.readup.com.cer;
    ssl_certificate_key /etc/ssl/dev.readup.com.key;
    ```
4. Update the nginx configuration file at `/opt/local/etc/nginx/nginx.conf` with the following changes:

    Set the server to run under your user account, otherwise you'll need to modify your directory permissions to allow the server process to access the static files:

	     user jeff staff;

    Add the Readup servers, replacing the directories to match the locations on your computer:

    ```
    http {
    	server {
    		server_name dev.api.readup.com;
    		location / {
    			proxy_pass http://127.0.0.1:5000;
    		}
    		include readup-server.conf;
    	}
    	server {
    		server_name dev.readup.com;
    		location / {
    			proxy_pass http://127.0.0.1:5001;
    		}
    		include readup-server.conf;
    	}
    	server {
    		server_name static.dev.readup.com;
    		location / {
    			root       /Users/jeff/readup/static/content;
    			add_header Access-Control-Allow-Origin *;
    			add_header Cache-Control "max-age=0";
    		}
        # Refers to the standalone static content repository (with image assets etc.)
        location / {
          root       /Users/jeff/readup/static/content;
          add_header Access-Control-Allow-Origin *;
          add_header Cache-Control "max-age=0";
        }
        # Refers to compiled web bundles in web
        location /app/bundles {
          alias      /Users/jeff/readup/web/bin/dev/app/client;
          add_header Access-Control-Allow-Origin *;
          add_header Cache-Control "max-age=0";
        }
    		location /common/auth-service-popup-handler/v1/index.html {
    			alias      /Users/jeff/readup/web/src/common/auth-service-popup-handler/index.html;
    			add_header Access-Control-Allow-Origin *;
    			add_header Cache-Control "max-age=0";
    		}
    		location /embed {
    			alias      /Users/jeff/readup/web/bin/dev/embed;
    			add_header Access-Control-Allow-Origin *;
    			add_header Cache-Control "max-age=0";
    		}
    		include readup-server.conf;
    	}
    	server {
    		server_name blog.dev.readup.com;
    		location / {
    			root       /Users/jeff/readup/blog/_site;
    			add_header Cache-Control "max-age=0";
    		}
    		include readup-server.conf;
    	}
    	server {
    		server_name article-test.dev.readup.com;
    		location / {
    			proxy_pass http://127.0.0.1:5002;
    		}
    		include readup-server.conf;
    	}
    	server {
    		server_name twitter-test.dev.readup.com;
    		location / {
    			proxy_pass http://127.0.0.1:5003;
    		}
    		include readup-server.conf;
    	}
    }
	 ```
5. Restart nginx to reload the configuration file.

        sudo nginx -s reload
6. If you haven't already done this, add the `ca.dev.reallyread.it.cer` certificate to your system as a trusted root certificate authority. Refer to the related instructions of the Docker setup, and take into account these considerations for developing with mobile devices.
    - iOS (Simulator)
	     - Drag and drop the `ca.dev.reallyread.it.cer` file into the Simulator window.
    - iOS (Physical Device)

	     - Copy `ios-profile.html` and `ca.dev.reallyread.it.cer` to the default nginx root directory: `/opt/local/share/nginx/html`
        - Navigate to the html file on your iOS device: `http://YOUR_COMPUTERS_IP/ios-profile.html`
        - Tap the link to the certificate and accept the prompt to create a new profile.
        - ... fill in rest of instructions when setting up with Bill.

7. Configure DNS: see the last steps of the Docker setup.

	 If you're developing on a physical iOS device you'll need to add DNS entries to your home router or whichever device is acting as the DNS server for your iOS device. In addition, iOS might complain if the DNS entry points to a local IP address. You can work around this by setting a second IP address on your development computer outside the subnet that the iOS device is on, pointing the DNS records at that address and configuring a static route to that address on your router.

## Other repositories
The following repositories can optionally be installed now or at any point in the future:

1. https://github.com/reallyreadit/article-test-server - A local source of articles so you can test Readup without interfacing with a 3rd party server. Contains articles with various content and metadata structures for parser testing.
2. https://github.com/reallyreadit/css-id - A small utility program for generating unique CSS class names for the purpose of scoping style rules to a specific component.
3. https://github.com/reallyreadit/ios - The Readup iOS client.
4. https://github.com/reallyreadit/twitter-test-server - A mock Twitter API implementation for integration testing.