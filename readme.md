Passing Real Client IP to a Service Behind CGNAT

This guide explains how to configure a self-hosted service (like Nextcloud) to see the real visitor IP address, even when your server is behind a Carrier-Grade NAT (CGNAT).

🎯 The Problem

When your home server is behind CGNAT, you can't expose it directly to the internet. A common solution is to use a VPS (Virtual Private Server) as a reverse proxy and connect it to your home server using a mesh VPN like Tailscale.

The problem is that your service (e.g., Nextcloud) will only see the IP address of your VPS (its Tailscale IP), not the original visitor's IP. This makes logging, monitoring, and security features like fail2ban difficult.

💡 The Solution

We will configure a chain of trust using the X-Forwarded-For HTTP header.

The VPS Reverse Proxy will capture the real client IP and place it in the X-Forwarded-For header.

The Home Server (Apache) will be configured to trust the VPS (via its Tailscale IP) and treat the IP in the X-Forwarded-For header as the true remote IP.

The Nextcloud Application will also be told to trust the VPS as a proxy.

Diagram of Network Topology

<img width="422" height="285" alt="image" src="https://github.com/user-attachments/assets/420fd127-8845-4012-a52c-30a4095c5b5d" />


[ Visitor (Public IP) ]
       |
       v
[ VPS (Public IP) ]
 (Reverse Proxy: Apache/Nginx)
   - Adds "X-Forwarded-For" header
       |
       v
[ Tailscale Network ]
       |
       v
[ Home Server (Behind CGNAT) ]
 (Apache + Nextcloud)
   - Trusts VPS Tailscale IP
   - Reads "X-Forwarded-For" header



📋 Prerequisites

Home Server: A server behind CGNAT running your service (e.g., Nextcloud on Apache).

VPS: A server with a public IP, acting as a reverse proxy (this guide assumes Apache).

Tailscale: Installed and running on both the Home Server and the VPS.

Working Proxy: You should already have a basic reverse proxy set up on the VPS that forwards a domain (e.g., nextcloud.your-domain.com) to your Home Server's Tailscale IP and port.

🚀 Configuration Steps

There are three parts to configure: The VPS (Proxy), the Home Server (Service), and the Application (Nextcloud).

Step 1: Configure the VPS (Reverse Proxy)

On your VPS, edit the Apache virtual host configuration file for your domain (e.g., /etc/apache2/sites-available/nextcloud.conf).

Inside your <VirtualHost> or <Location> block for the proxy, add the following RequestHeader directives. This takes the actual visitor's IP (%{REMOTE_ADDR}s) and puts it into the X-Forwarded-For header.

# /etc/apache2/sites-available/nextcloud.conf
```
<VirtualHost *:443>
    ServerName nextcloud.your-domain.com
    
    # --- ADD THESE THREE LINES ---
    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Host "%{HTTP_HOST}s"
    RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s"
    
    # Your existing proxy config (example)
    ProxyPreserveHost On
    ProxyPass / http://[HOME_SERVER_TAILSCALE_IP]:80/
    ProxyPassReverse / http://[HOME_SERVER_TAILSCALE_IP]:80/
    
    # Your SSL Config (Certbot, etc.)
    #
</VirtualHost>
```

After saving, restart Apache on the VPS:

sudo systemctl restart apache2


Step 2: Configure the Home Server (Apache)

On your Home Server, we need to tell Apache to trust the X-Forwarded-For header only when it comes from your VPS.

First, find your VPS's Tailscale IP. On the VPS (or any machine in the tailnet), run:

tailscale status


Find the IP for your VPS (e.g., 100.x.x.x).

Create a new Apache configuration file to enable mod_remoteip:
```
sudo nano /etc/apache2/conf-available/remoteip.conf
```

Add the following content. This tells mod_remoteip to look for the X-Forwarded-For header and to trust the proxy providing it (your VPS).

# /etc/apache2/conf-available/remoteip.conf
```
# 1. Tell Apache to use the X-Forwarded-For header
RemoteIPHeader X-Forwarded-For

# 2. Tell Apache to trust your VPS's Tailscale IP
#    (Replace 100.x.x.x with your VPS's Tailscale IP)
RemoteIPTrustedProxy 100.x.x.x
```

Enable the new configuration and the remoteip module:
```
sudo a2enconf remoteip
```
```
sudo a2enmod remoteip
```

Restart Apache on your Home Server:
```
sudo systemctl restart apache2
```

Step 3: Configure the Nextcloud Application

Finally, tell the Nextcloud application itself to trust the proxy.

Edit your Nextcloud config.php file, usually located at /var/www/nextcloud/config/config.php.

Add your VPS's Tailscale IP to the trusted_proxies array.
```
<?php
$CONFIG = [
  // ... other config options

  'trusted_proxies' => [
    '100.x.x.x', // <-- Add your VPS's Tailscale IP here
  ],

  // ... other config options
];
```

✅ Verification

You're all done! After restarting all services, the X-Forwarded-For header from the proxy will be passed to your Nextcloud server. Apache will use it to overwrite the request's IP, and Nextcloud will trust it.

To check if it's working, log in to your Nextcloud instance and go to Settings -> Administration -> Logging. You should now see log entries from your real public IP address instead of your VPS's Tailscale IP.
