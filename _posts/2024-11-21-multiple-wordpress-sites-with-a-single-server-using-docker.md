---
published: true
date: 2024-01-03
title: Multiple Wordpress Sites with a single server using Docker
---
Do you need a website or two, or three? While you can just go over to [Wordpress.org](http://Wordpress.org) and have multiple sites, it’s pretty limited functionality wise — no plugins unless you pay what I think is a ridiculous amount of money and ditto for custom domains. It kind of sucks.

The next step up is going to a web host and getting a shared cpanel instance, which is pretty easy to use, and installing Wordpress on it. Pretty simple stuff. You can go over to sites like Hostgator and get a site for $3.95 this way, and be able to use a custom domain name, and install the full version of Wordpress. This is the easiest way to get a single full featured Wordpress instance for cheap, but running multiple sites on an entry level shared hosting plan isn’t ideal.

I wanted to host multiple Wordpress sites and have room to run other services, so I went in a slightly more complicated direction. I got a VPS (virtual private server) from Contabo — for $6.99 a month you can get a VPS with 4 vCPU’s, 8GB of ram, and a 50 GB NMVe ssd. That’s pretty freakin awesome for just $7 a month! I can pretty much host as many wordpress sites as I want. And by containerizing my wordpress sites with docker, if one them falls victim to a compromise, it’ll be isolated from the other ones and I can run different versions if need be. I can also restart them independently and they won’t share a database.

Beats the hell out of shared hosting!

It’s a little more hands on than shared hosting and a lot more hands on than [wordpress.org](http://wordpress.org), but its still pretty easy to manage. I’m going to go through accessing a VPS from SSH, installing docker and docker compose, NGINx proxy manager, setting up DNS and a the Wordpress containers. I use a VPS but you could easily do the same thing on a local server or dedicated server.

First off, buy a VPS from somewhere, it doesn’t really matter where. Some hosts are better than others in regards to customer support and some have much better pricing than others.

Secondly, get your domains. Domain registrars are a dime a dozen, but once again, some are better than others. I like namecheap because their domains come with free DNS privacy, whereas somewhere like godaddy charges extra. Personal preference here.

Once you purchase your vps, you’ll get your SSH login credentials to access the server. Next you’ll open up the terminal if you’re on a Mac or linux, or download on SSH client if your on windows. All you have to do is type:

```
ssh username@127.0.0.1
```

Obviously substituting the provided username and IP address, and then entering the password you either set, or that they also provided.

Our Nginx proxy manager and our Wordpress site will both be hosted in docker containers, so the next step is to install docker. I’m using Ubuntu for my VPS so the exact commands might not be the same for you, but the gist of it will be the same.

# **Installing Docker**

First we’ll update the system. apt-get update see’s if there’s updates for the packages on the system, and apt-get upgrade actually installs them.

```
sudo apt-get update && sudo apt-get upgrade
```

Now we can install the prerequisites for docker and add the repository, since it isn’t in the standard Ubuntu repositories.

```
sudo apt-get install \
ca-certificates \
curl \
gnupg \
lsb-release
```

The below command adds dockers GPG key. The GPG key makes sure we know we’re installing a signed, official version of docker when we download from the repository.

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg  - dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

Then we add the docker repository:

```
echo \
"deb [arch=$(dpkg  - print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Run sudo apt-get update one last time to load the new repository and we can finally install docker.

```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Make sure docker is installed correctly with sudo docker run hello-world.

You should get output that looks like this:

<img src="https://miro.medium.com/v2/resize:fit:1400/1*QBNYuHPEr5cyU_r_QTfxtw.png" alt="" class="m yj akd c" style="box-sizing: inherit; vertical-align: middle; background-color: rgb(255, 255, 255); width: 680px; max-width: 100%; height: auto;" width="700" height="447">

Making sure docker works by pulling a hello world image down

# **Install Nginx Proxy Manager**

Nginx proxy manager is a web based tool that acts as a reverse proxy. Now, what is a reverse proxy and why do we need it?

Our server has 1 IP address. If we want to host multiple websites on our server, we need a way to use a single IP with multiple websites. A reverse proxy allows us to do! It takes an incoming connection to the server, and forwards it to the correct port. For example, lets say we have websites A, B and C all running on a server with a public IP of 162.159.153.4. Internally, these 3 websites are running on ports 80, 81, and 82 respectively.

In DNS, each of the sites are pointed at the same IP, yet we are able to access all 3 individually. How? The reverse proxy takes the incoming request, looks at what domain name is being requested, and points the request to the correct port. So if we are accessing [B.com](http://B.com), the reverse proxy will know to point us at port 81, and so on.

We will be setting up Nginx in a docker container, using docker compose. I’m ripping the code right from [https://nginxproxymanager.com](https://nginxproxymanager.com)[.](https://nginxproxymanager.com./)

We’ll create a folder “apps” to hold all of our docker-compose scripts, and a folder for nginxproxymanager inside that folder.

```
mkdir apps
cd apps
mkdir nginxproxymanager
cd nginxproxymanager
```

And create a docker-compose file

```
nano docker-compose.yml
```

And just copy this in into the file, we don’t need to change the ports or anything.

```
version: "3"
services:
  app:
  image: 'jc21/nginx-proxy-manager:latest'
  restart: unless-stopped
  ports:
  # These ports are in format <host-port>:<container-port>
  - '80:80' # Public HTTP Port
  - '443:443' # Public HTTPS Port
  - '81:81' # Admin Web Port
  # Add any other Stream port you want to expose
  # - '21:21' # FTP

  # Uncomment the next line if you uncomment anything in the section
  # environment:
  # Uncomment this if you want to change the location of
  # the SQLite DB file within the container
  # DB_SQLITE_FILE: "/data/database.sqlite"

  # Uncomment this if IPv6 is not enabled on your host
  # DISABLE_IPV6: 'true'

  volumes:
  - ./data:/data
  - ./letsencrypt:/etc/letsencrypt
```

Save with ctrl + x and run `docker-compose up -d` to start the container. Go to a web browser and type in the address of your server like so: 127.0.0.0:81.

Voila!

<img src="https://miro.medium.com/v2/resize:fit:1400/1*KOA8HObseSkTGLWZupZBmQ.png" alt="" class="m yj akd c" style="box-sizing: inherit; vertical-align: middle; background-color: rgb(255, 255, 255); width: 680px; max-width: 100%; height: auto;" width="700" height="364">

Log in using the default credentials [admin@example.com](mailto:admin@example.com) and password “changeme” and it will prompt you to create new credentials. We will come back to this after installing Wordpress.

# **Get a wordpress container**

Change directory back up into apps, make a folder for your site, and cd into it. Make another docker compose file (make sure it’s also docker-compose.yml)and paste in the below.

```
version: "3.9"
services:
  db:
  image: mysql:5.7
  volumes:
  - db_data:/var/lib/mysql
  restart: always
  environment:
  MYSQL_ROOT_PASSWORD: somewordpress
  MYSQL_DATABASE: wordpress
  MYSQL_USER: wordpress
  MYSQL_PASSWORD: wordpress
 
  wordpress:
  depends_on:
  - db
  image: wordpress:latest
  volumes:
  - wordpress_data:/var/www/html
  ports:
  - "8000:80"
  restart: always
  environment:
  WORDPRESS_DB_HOST: db
  WORDPRESS_DB_USER: wordpress
  WORDPRESS_DB_PASSWORD: wordpress
  WORDPRESS_DB_NAME: wordpress
volumes:
  db_data: {}
  wordpress_data: {}
```

In this compose file, the port number is important. It will map port 8000 on the server to port 80 in the docker container.

If you want to create another site on the same server, repeat the steps above in a new directory, paste in the same code above and change the outside port to something else, like 8001.

Run `docker-compose up -d` to start the container up, it will be accessible at your server ip and port, like this: 127.0.0.1:8000.

DNS

Head over to where you registered your domain to set up an A record pointed at your server IP. You’ll set the host to “@” and the value to your instances IP address. If you plan on having multiple domains going to multiple Wordpress instances, they will all point at the same IP — the proxy manager we installed will take care of directing traffic to the appropriate container.

# **Final Steps**

Head back into nginx proxy manager, login and head to hosts < proxy hosts < add proxy host. Add your domain name, the ip of your server and the port we set up earlier. You’d do the same thing for each subsequent Wordpress instance as well, if hosting multiple.

<img src="https://miro.medium.com/v2/resize:fit:1400/1*NpksWVRlREZmyED18z_uxg.png" alt="" class="m yj akd c" style="box-sizing: inherit; vertical-align: middle; background-color: rgb(255, 255, 255); width: 680px; max-width: 100%; height: auto;" width="700" height="784">

Then go to the SSL tab, and request a new SSL certificate. And we’re done! Easy peasy.

Now you can head over to your domain, which is now connected to your freshly dockerized Wordpress site!