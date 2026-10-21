---
published: true
date: 2024-01-03
title: Multiple Wordpress Sites with a single server using Docker
---
Do you need a website or two, or three? While you can just go over to [WordPress.com](https://wordpress.com) and have multiple sites, it’s pretty limited functionality wise — no plugins unless you pay what I think is a ridiculous amount of money and ditto for custom domains. It kind of sucks.

The next step up is going to a web host and getting a shared cpanel instance, which is pretty easy to use, and installing Wordpress on it. Pretty simple stuff. You can go over to sites like Hostgator and get a site for $3.95 this way, and be able to use a custom domain name, and install the full version of Wordpress. This is the easiest way to get a single full featured Wordpress instance for cheap, but running multiple sites on an entry level shared hosting plan isn’t ideal.

I wanted to host multiple Wordpress sites and have room to run other services, so I went in a slightly more complicated direction. I got a VPS (virtual private server) from Contabo — for $6.99 a month you can get a VPS with 4 vCPU’s, 8GB of ram, and a 50 GB NVMe ssd. That’s pretty freakin awesome for just $7 a month! I can pretty much host as many wordpress sites as I want. And by containerizing my wordpress sites with docker, if one them falls victim to a compromise, it’ll be isolated from the other ones and I can run different versions if need be. I can also restart them independently and they won’t share a database.

Beats the hell out of shared hosting!

It’s a little more hands on than shared hosting and a lot more hands on than [WordPress.com](https://wordpress.com), but its still pretty easy to manage. I’m going to go through accessing a VPS from SSH, installing docker and docker compose, NGINx proxy manager, setting up DNS and the Wordpress containers. I use a VPS but you could easily do the same thing on a local server or dedicated server.

First off, buy a VPS from somewhere, it doesn’t really matter where. Some hosts are better than others in regards to customer support and some have much better pricing than others.

Secondly, get your domains. Domain registrars are a dime a dozen, but once again, some are better than others. I like namecheap because their domains come with free DNS privacy, whereas somewhere like godaddy charges extra. Personal preference here.

Once you purchase your vps, you’ll get your SSH login credentials to access the server. Next you’ll open up the terminal if you’re on a Mac or linux, or download an SSH client if you’re on windows. All you have to do is type:

```
ssh username@your.server.ip
```

Obviously substituting the provided username and IP address, and then entering the password you either set, or that they also provided.

Our Nginx proxy manager and our Wordpress site will both be hosted in docker containers, so the next step is to install docker. I’m using Ubuntu for my VPS so the exact commands might not be the same for you, but the gist of it will be the same.

# **Installing Docker**

First we’ll update the system. apt-get update sees if there are updates for the packages on the system, and apt-get upgrade actually installs them.

```
sudo apt-get update && sudo apt-get upgrade
```

Now we can install the prerequisites for docker and add the repository, since it isn’t in the standard Ubuntu repositories.

```
sudo apt-get install ca-certificates curl gnupg lsb-release
```

The below command adds Docker’s GPG key. The GPG key makes sure we know we’re installing a signed, official version of docker when we download from the repository.

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

Then we add the docker repository:

```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Run apt-get update one last time to load the new repository and we can finally install docker.

```
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Note that the compose plugin is run as `docker compose` (with a space), not the older `docker-compose`.

Make sure docker is installed correctly with `sudo docker run hello-world`.

You should get output that looks like this:

![Output of docker run hello-world](https://miro.medium.com/v2/resize:fit:1400/1*QBNYuHPEr5cyU_r_QTfxtw.png)

Making sure docker works by pulling a hello world image down

# **Install Nginx Proxy Manager**

Nginx proxy manager is a web based tool that acts as a reverse proxy. Now, what is a reverse proxy and why do we need it?

Our server has 1 IP address. If we want to host multiple websites on our server, we need a way to use a single IP with multiple websites. A reverse proxy allows us to do that! It takes an incoming connection to the server and forwards it to the correct site. For example, let’s say we have websites A, B and C all running on a server with a public IP of 162.159.153.4. Internally, these 3 websites are running in separate containers.

In DNS, each of the sites are pointed at the same IP, yet we are able to access all 3 individually. How? The reverse proxy takes the incoming request, looks at what domain name is being requested, and points the request to the correct container. So if we are accessing B.com, the reverse proxy will know to send us to site B, and so on.

All of the containers will talk to each other over a shared docker network, so the WordPress sites never need to be exposed to the internet directly. Only the proxy is. Create that network once:

```
sudo docker network create proxy
```

We will be setting up Nginx in a docker container, using docker compose. The code is adapted from [nginxproxymanager.com](https://nginxproxymanager.com).

We’ll create a folder “apps” to hold all of our docker compose files, and a folder for nginxproxymanager inside that folder.

```
mkdir -p apps/nginxproxymanager
cd apps/nginxproxymanager
```

And create a docker compose file

```
nano docker-compose.yml
```

And copy this into the file:

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      # These ports are in format <host-port>:<container-port>
      - '80:80'   # Public HTTP Port
      - '443:443' # Public HTTPS Port
      - '127.0.0.1:81:81' # Admin Web Port, only reachable from the server itself
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    networks:
      - proxy

networks:
  proxy:
    external: true
```

The admin port is bound to 127.0.0.1 on purpose. Docker publishes ports straight through the firewall (it bypasses ufw), so if you publish port 81 normally, the admin login page is open to the whole internet.

Save with ctrl + o, exit with ctrl + x, and run `sudo docker compose up -d` to start the container. To reach the admin page, open an SSH tunnel from your own computer:

```
ssh -L 8081:127.0.0.1:81 username@your.server.ip
```

Then go to a web browser and type in http://localhost:8081.

Voila!

![Nginx Proxy Manager login page](https://miro.medium.com/v2/resize:fit:1400/1*KOA8HObseSkTGLWZupZBmQ.png)

Log in using the default credentials admin@example.com and password “changeme” and it will prompt you to create new credentials. We will come back to this after installing Wordpress.

# **Get a wordpress container**

Change directory back up into apps, make a folder for your site, and cd into it. Make another docker compose file (make sure it’s also docker-compose.yml) and paste in the below. Change the passwords, and change `mysite` to something unique for each site.

```yaml
services:
  db:
    image: mysql:8.4
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: change-this-root-password
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: change-this-password

  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    container_name: mysite
    volumes:
      - wordpress_data:/var/www/html
    restart: always
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: change-this-password
      WORDPRESS_DB_NAME: wordpress
    networks:
      - default
      - proxy

volumes:
  db_data: {}
  wordpress_data: {}

networks:
  proxy:
    external: true
```

Notice there’s no `ports:` section. The WordPress container joins the `proxy` network, so Nginx Proxy Manager can reach it by its container name, and nothing else can. The database stays on the site’s own private network and only WordPress can talk to it.

If you want to create another site on the same server, repeat the steps above in a new directory, paste in the same code, and give it a different `container_name` (like `mysite2`). Each site gets its own database.

Run `sudo docker compose up -d` to start the containers up.

# **DNS**

Head over to where you registered your domain to set up an A record pointed at your server IP. You’ll set the host to “@” and the value to your instance’s IP address. If you plan on having multiple domains going to multiple Wordpress instances, they will all point at the same IP. The proxy manager we installed will take care of directing traffic to the appropriate container.

# **Final Steps**

Head back into nginx proxy manager, log in and go to Hosts > Proxy Hosts > Add Proxy Host. Add your domain name, set the scheme to http, the forward hostname to your container name (`mysite`), and the forward port to 80. You’d do the same thing for each subsequent Wordpress instance as well, if hosting multiple. (The screenshot below is from the original version of this post, which used the server IP and port instead.)

![Adding a proxy host in Nginx Proxy Manager](https://miro.medium.com/v2/resize:fit:1400/1*NpksWVRlREZmyED18z_uxg.png)

Then go to the SSL tab, and request a new SSL certificate. And we’re done! Easy peasy.

Now you can head over to your domain, which is now connected to your freshly dockerized Wordpress site!
