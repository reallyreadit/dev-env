# Readup Development Environment Setup Guide
## Repositories
### Prerequisites
Readup will run on Linux, macOS or Windows. The guides in the readme files use macOS commands and directory structures as examples so translations may be required for Linux and Windows systems. PowerShell is required to execute some scripts that are included in various repositories. It is included by default in Windows but will need to be installed separately on Linux and macOS systems: https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell
### Required
There are three main repositories that constitute the core of the Readup platform: `db`, `api` and `web`. Begin setting up the development environment by cloning these repositories and following their respective readme files in the order listed:

1. https://github.com/reallyreadit/db
2. https://github.com/reallyreadit/api
3. https://github.com/reallyreadit/web

Once completed, you'll have database, API and web servers running locally. You could technically run Readup at this point but the sample configuration files use `*.dev.readup.com` domain names and `https` for each service because browsers treat websites running on `localhost` or an IP address and served over non-default ports or an insecure connection with special consideration. In order to more closely match the production environment we'll set up a reverse-proxy web server in a following step.
### Optional
The following repositories can optionally be installed now or at any point in the future:

1. https://github.com/reallyreadit/article-test-server - A local source of articles so you can test Readup without interfacing with a 3rd party server. Contains articles with various content and metadata structures for parser testing.
2. https://github.com/reallyreadit/blog - The Readup blog. The reverse-proxy server can be configured to point to the build output directory. Useful for testing the embed.
3. https://github.com/reallyreadit/css-id - A small utility program for generating unique CSS class names for the purpose of scoping style rules to a specific component.
4. https://github.com/reallyreadit/ios - The Readup iOS client.
5. https://github.com/reallyreadit/twitter-test-server - A mock Twitter API implementation for integration testing.
## Static Content
The `static.readup.com` production server serves four main purposes:

1. **Hosting font and image files used by the various Readup clients.**
    
	 These files are included in this repository in the `static` directory. This will be the root directory of our `static.readup.com` development server.
2. **Hosting the authentication service popup handler HTML file.**

    This is a single static source file in the `web` repository that is not currently handled by the build system. We'll set up an alias at `/common/auth-service-popup-handler/v1/index.html` that points to the source file.
3. **Hosting the embed script files.**

    These files are generated by the `web` repository build system. We'll set up a virtual directory at `/embed` that points to the build output directory.
4. **Hosting extension and native client script files and indexes of those files in order to facilitate over-the-air updates.**

    Setup of these directories is currently a manual process not covered in this guide. Not required for development use except for testing of the script update functionality.
## Reverse-Proxy Web Server
This setup guide uses nginx but any web server capable of acting as a reverse-proxy and serving static content will work. The directories and commands are for macOS and will need to be modified for other operating systems.
1. Install nginx using Macports or Homebrew
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
    			add_header Cache-Control "max-age=0";
    		}
    		location /common/auth-service-popup-handler/v1/index.html {
    			alias      /Users/jeff/readup/web/src/common/auth-service-popup-handler/index.html;
    			add_header Cache-Control "max-age=0";
    		}
    		location /embed {
    			root       /Users/jeff/readup/web/bin/dev/embed;
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

        sudo /opt/local/sbin/nginx -s reload
6. Add the `ca.dev.reallyread.it.cer` certificate to your system as a trusted root certificate authority.
    
	 - macOS

	     - Import into "System" in Keychain Access
		  - Right-click certificate and select "Get Info"
		  - Expand the "Trust" section and select "Always Trust"
    - iOS

	     - Copy `ios-profile.html` and `ca.dev.reallyread.it.cer` to the default nginx root directory: `/opt/local/share/nginx/html`
        - Navigate to the html file on your iOS device: `http://YOUR_COMPUTERS_IP/ios-profile.html`
        - Tap the link to the certificate and accept the prompt to create a new profile.
        - ... fill in rest of instructions when setting up with Bill.
	 - Windows

	     - Import into the local computer's "Trusted Root Certification Authorities" location.
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