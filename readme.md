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

The easiest way to use [Docker](https://www.docker.com/get-started/) and our Docker Compose configuration. With one command, this will install and run the required server containers to get started: the node.js 14 `web` server, ASP.NET 3.1 `api` server, Postgres 14 `db` server, and a reverse proxy server that provides a HTTPS interface to the previous services. See "Install via Docker" below.

Alternatively, you can manually install all these dependencies by following the readmes of [`db`](https://github.com/reallyreadit/db), [`api`](https://github.com/reallyreadit/api) and [`web`](https://github.com/reallyreadit/web), in that order. Then skip to "Manual Installation" in readme.


## Quickstart: install via Docker

### 1. Run the containers with Docker Compose

Run this command in the the root of this `dev-env`. Make sure that the `web`, `dev` and `api` repository folders are siblings of the `dev-env` folder, otherwise it will not work. 

```
docker-compose -p readup -f docker-compose.yml up -d
```

This command will download, install, configure and run the node.js 14 `web` server, ASP.NET `api` server, Postgres 14 `db` server, and a reverse proxy server.

With the default Dockerfiles, `web` will run a development server that can be used for basic web app development purposes. To develop the Readup embed, web extension, desktop or iOS app, you need to read the docs of the `web` and other relevant repositories (`ios`, `desktop`) to learn how to further configure your environment, and which build commands to run inside the container (some Docker experience is helpful).

### 2. Import a database dump

Development won't go smoothly with an empty database, you'll need to seed in a development database dump.

**NOTE:** we do not publicly provide a database dump right now, but you can request one in our Discord #development channel.

Add `my-database-dump.tar` to the `db` repo folder. Then run the following command from within the `db` repo to import it:

```
docker exec --user postgres readup_db_1 pwsh /dev-scripts/restore.ps1 -DbName rrit -DumpFile /host/my-database-dump.tar
```

In this command, `readup_db_1` is the name of your db container. Verify your container also has this name with `docker container ls`.


### 3. Trust Readup's SSL root certificate authority

You could technically run Readup at this point, but the default web server configuration makes Readup run via `https` (and not `http`), because browsers treat websites served over an insecure connection with special consideration. To enable HTTPS for development purposes, Readup uses a self-signed certificate included in this `dev-env` repository. 

In order to make sure that your system accepts this self-signed ceritficate, `ca.dev.reallyread.it.cer` from the `ssl` directory in this repo should be added to your system as a trusted root certificate authority. Here's how:
    
- macOS
	- Import into "System" in Keychain Access
	- Right-click certificate and select "Get Info"
	- Expand the "Trust" section and select "Always Trust"
- Windows: import into the local computer's "Trusted Root Certification Authorities" location.

On macOS and Windows most browsers will accept the certificate once added to the system certificate store. Firefox on macOS at least uses its own certificate store and requires the following steps. Other browsers may be similar.
1. Open "Preferences" and go to "Privacy & Security"
2. Scroll down to the "Certificates" section and click "View Certificates..."
3. Go to the "Authorities" tab and click "Import" to import the `ca.dev.reallyread.it.cer` certificate.

### 4. Set up your hosts file

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
### 5. Done!?

You should now be able to navigate to [https://dev.readup.com](https://dev.readup.com) in your browser, and see the Readup homepage.

Running [https://dev.readup.com/?clientType=App](https://dev.readup.com/?clientType=App) should allow you to create a new Readup account in your development environment and open the Readup web app.
## Manual Installation

Refer to this section when you can not (or don't want to) use Docker, or when you want to understand the inner workings of the Readup dev env for more advanced configuration.

This section assumes that you have already worked through the following the readmes of [`db`](https://github.com/reallyreadit/db), [`api`](https://github.com/reallyreadit/api) and [`web`](https://github.com/reallyreadit/web), in that order.

 The guides in the readme files use macOS commands and directory structures as examples so translations may be required for Linux and Windows systems. PowerShell is required to execute some scripts that are included in various repositories. It is included by default in Windows but will need to be installed separately on Linux and macOS systems: https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell

Once you completed manual set-up instructions of readmes, you'll have database, API and web servers running locally. For the reasons outlined in the Docker installation steps 4 & 5, you'll need to manually set up a reverse proxy that allows us to host the Readup web app at `https://dev.readup.com`, locally. That reverse proxy also

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
    			root       /Users/jeff/readup/dev-env/static;
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
    			root       /Users/jeff/readup/web/bin/dev/embed;
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
2. https://github.com/reallyreadit/blog - The Readup blog. The reverse-proxy server can be configured to point to the build output directory. Useful for testing the embed.
3. https://github.com/reallyreadit/css-id - A small utility program for generating unique CSS class names for the purpose of scoping style rules to a specific component.
4. https://github.com/reallyreadit/ios - The Readup iOS client.
5. https://github.com/reallyreadit/twitter-test-server - A mock Twitter API implementation for integration testing.