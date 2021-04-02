# Readup Development Environment Setup Guide
## Repositories
### Prerequisites
Readup will run on Linux, macOS or Windows. The guides in the readme files use macOS commands and directory structures as examples so translations may be required for Linux and Windows systems. PowerShell is required to execute some scripts that are included in various repositories. It is included by default in Windows but will need to be installed separately on Linux and macOS systems: https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell
### Required
There are four main repositories that constitute the core of the Readup platform: `db`, `api`, `web` and `static`. Begin setting up the development environment by cloning these repositories and following their respective readme files in the order listed:

1. https://github.com/reallyreadit/db
2. https://github.com/reallyreadit/api
3. https://github.com/reallyreadit/web
4. https://github.com/reallyreadit/static

Once completed, you'll have database, API and web servers running locally but the reverse-proxy web server described below must be installed and configured before everything will work properly. In order to more closely match the structure of the production environment the sample configuration files use `*.dev.readup.com` domain names and `https` for each service because browsers treat websites running on `localhost` or an IP address and served over non-default ports or an insecure connection with special consideration. Additionally we must configure `static.readup.com` to serve static content from the `static` repository and build output from the `web` repository.
### Optional
The following repositories can optionally be installed now or at any point in the future:

1. https://github.com/reallyreadit/article-test-server - A local source of articles so you can test Readup without interfacing with a 3rd party server. Contains articles with various content and metadata structures for parser testing.
2. https://github.com/reallyreadit/blog - The Readup blog. The reverse-proxy server can be configured to point to the build output directory. Useful for testing the embed.
3. https://github.com/reallyreadit/css-id - A small utility program for generating unique CSS class names for the purpose of scoping style rules to a specific component.
4. https://github.com/reallyreadit/ios - The Readup iOS client.
5. https://github.com/reallyreadit/twitter-test-server - A mock Twitter API implementation for integration testing.
## Reverse-Proxy Web Server
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
    		location /app/bundles {
    			alias      /Users/jeff/readup/web/bin/dev/app/client;
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
6. Add the `ca.dev.reallyread.it.cer` certificate to your system as a trusted root certificate authority.

	 - macOS

	     - Import into "System" in Keychain Access
		  - Right-click certificate and select "Get Info"
		  - Expand the "Trust" section and select "Always Trust"
    - iOS (Simulator)
	     - Drag and drop the `ca.dev.reallyread.it.cer` file into the Simulator window.
    - iOS (Physical Device)

	     - Copy `ios-profile.html` and `ca.dev.reallyread.it.cer` to the default nginx root directory: `/opt/local/share/nginx/html`
        - Navigate to the html file on your iOS device: `http://YOUR_COMPUTERS_IP/ios-profile.html`
        - Tap the link to the certificate and accept the prompt to create a new profile.
        - ... fill in rest of instructions when setting up with Bill.
	 - Windows

	     - Import into the local computer's "Trusted Root Certification Authorities" location.

    Note: On macOS and Windows most browsers will accept the certificate once added to the system certificate store. Firefox on macOS at least uses its own certificate store and requires the following steps. Other browsers may be similar.
	 1. Open "Preferences" and go to "Privacy & Security"
	 2. Scroll down to the "Certificates" section and click "View Certificates..."
	 3. Go to the "Authorities" tab and click "Import" to import the `ca.dev.reallyread.it.cer` certificate.
7. Configure DNS.

    If you're developing on macOS or Windows you only need to add the following entries to your hosts file located at `/private/etc/hosts` or `C:\Windows\System32\drivers\etc\hosts` respectively:

    ```
    127.0.0.1 api.dev.readup.com
    127.0.0.1 dev.readup.com
    127.0.0.1 static.dev.readup.com
    127.0.0.1 blog.dev.readup.com
    127.0.0.1 article-test.dev.readup.com
    127.0.0.1 twitter-test.dev.readup.com
    ```

	 If you're developing on a physical iOS device you'll need to add DNS entries to your home router or whichever device is acting as the DNS server for your iOS device. In addition, iOS might complain if the DNS entry points to a local IP address. You can work around this by setting a second IP address on your development computer outside the subnet that the iOS device is on, pointing the DNS records at that address and configuring a static route to that address on your router.