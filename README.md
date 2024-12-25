# Nominatim 4.5.0 Setup for OpenStreetMap: Nepal Map Data

###### Date: 24-Dec-2024 <br>

###### Version: 1.0 </br>

#### [shreeniraula.com.np](https://shreeniraula.com.np/) | LinkedIn: [ShreeNiraula](https://www.linkedin.com/in/shreeniraula/) | Twitter: [@ShreeNiraula](https://twitter.com/ShreeNiraula) | YouTube: [@ShreeNiraula](https://www.youtube.com/@ShreeNiraula/)

Welcome! This repository walks you through setting up Nominatim, a powerful geocoding tool that works with OpenStreetMap (OSM) data. This guide is specifically designed for Nepal, making it easy to set up and start using OSM data for geospatial services like location search, reverse geocoding, and more.

The setup includes everything you need, from installing dependencies to configuring PostgreSQL, setting up Nginx as a reverse proxy, and securing your server with HTTPS. Once set up, you'll have a fully functional Nominatim instance for your own geospatial needs.

## Why Use Nominatim?

Here are some key reasons why Nominatim is a great choice for your geocoding needs:

- **Completely Free**: Nominatim is open-source, meaning you don't have to pay any licensing fees to use it, and you can modify it to suit your needs.
- **Accurate and Up-to-Date**: It uses OpenStreetMap, which is constantly updated by contributors around the world, ensuring that the geographic data you’re working with is accurate and current.
- **Highly Customizable**: You can tweak the setup to focus on specific regions or change configurations to meet the needs of your application.
- **Privacy-First**: By hosting your own instance of Nominatim, you maintain complete control over your data, offering you peace of mind in terms of privacy.
- **Built for Scale**: Whether you're running a small app or handling a large number of requests, Nominatim is designed to scale easily, making it suitable for any project.

## Nepali App Examples Using Geocoding:

- **Tootle**: A popular Nepali ride-hailing app that uses geocoding for locating riders and drivers, optimizing travel routes, and managing pick-up points.
- **Pathao Nepal**: Used for ride services and deliveries, Pathao employs geocoding to find the user's current location and their destination for accurate route mapping.
- **Hamro Patro**: A Nepali app offering calendar, news, and other local services that might use location-based features for festivals, local events, and nearby places.

## Global Apps Using Geocoding:

- **Uber**: Uses geocoding for ride request locations and destination searches.
- **Airbnb**: Relies on geocoding to help users search for accommodations by location.
- **Foursquare**: Uses location data for recommending nearby places to eat, shop, or visit.

## Getting Started

If you're ready to set up your own Nominatim instance, just follow the steps in this repository. Here's a quick overview of what you'll be doing:

1. **Install Dependencies**: We'll walk you through installing PostgreSQL, PostGIS, osm2pgsql, and the necessary Python packages.
2. **Set Up PostgreSQL**: We'll create the necessary users and databases for Nominatim and configure PostgreSQL.
3. **Import OSM Data**: We provide steps to download and import the latest OpenStreetMap data for Nepal into your Nominatim database.
4. **Configure the Server**: Set up Nginx as a reverse proxy and secure the server with HTTPS.
5. **Run Nominatim**: Finally, we guide you through running Nominatim as a Gunicorn application to make it available to your users.

Follow the full instructions in this repository, and you'll have a fully functional Nominatim setup in no time!

## Prerequisites

Before you begin, ensure your machine meets the following requirements:

- **Debian-based operating system (e.g., Ubuntu)**: This guide assumes you're using a Debian-based distribution like Ubuntu. Other Linux distributions may require different commands or configurations.
- **Elevated privileges (sudo access)**: You'll need administrator (root) access to install software, configure system settings, and manage users. Ensure you have `sudo` privileges on your machine to carry out these actions.
- **Set the A Record for the domain name (optional)**

##

### Map the domain name to localhost by editing the hosts file (Optional)

`sudo vi /etc/hosts`

`127.0.0.1 osm.shreeniraula.com.np`

## Nominatim Installation & Configuration

#### Install necessary packages for PostgreSQL, PostGIS, osm2pgsql, and Python dependencies

First, update your package list and install the required dependencies:

```bash
sudo apt-get install -y osm2pgsql postgresql-postgis postgresql-postgis-scripts \
pkg-config libicu-dev virtualenv python3-pip acl
```

#### Create a system user 'nominatim' and set up its home directory

```bash
sudo useradd -d /srv/nominatim -s /bin/bash -m nominatim
sudo mkdir -p /srv/nominatim
sudo chown -R nominatim:nominatim /srv/nominatim
```

Switch to the 'nominatim' user to continue with setup

```bash
sudo -u nominatim bash
cd
```

Set environment variables for the 'nominatim' user

```bash
export USERNAME=nominatim
export USERHOME=/srv/nominatim
chmod a+x $USERHOME
```

Switch to Admin user and create PostgreSQL users for Nominatim and web service

```bash
sudo -u postgres createuser nominatim
sudo -u postgres createuser www-data
```

Grant 'nominatim' user permission to create databases in PostgreSQL

```bash
sudo -u postgres psql
ALTER USER nominatim WITH CREATEDB;
CREATE EXTENSION IF NOT EXISTS postgis; # Enable PostGIS extension
\du # List PostgreSQL roles
\q # Exit PostgreSQL prompt
```

Switch back to nominatim user and create a Python virtual environment for Nominatim

```bash
virtualenv $USERHOME/nominatim-venv
```

Install the latest version of Nominatim using pip

```bash
$USERHOME/nominatim-venv/bin/pip install nominatim-db
```

Activate the virtual environment to run Nominatim commands

```bash
. $USERHOME/nominatim-venv/bin/activate
```

Create the Nominatim project directory and navigate into it, setting an environment variable for the directory path

```bash
mkdir ~/nominatim-project
cd ~/nominatim-project
export PROJECT_DIR=~/nominatim-project
cd $PROJECT_DIR
```

Download the OSM data for Nepal from Geofabrik

```bash
wget https://download.geofabrik.de/asia/nepal-latest.osm.pbf
```

Switch to admin user and grant 'nominatim' user superuser privileges temporarily to import data

```bash
sudo -u postgres psql
ALTER USER nominatim WITH SUPERUSER;
\q
```

Switch to 'nominatim' user and import the OSM data into the Nominatim database

```bash
nominatim import --osm-file nepal-latest.osm.pbf 2>&1 | tee setup.log
```

Verify the Nominatim database setup

```bash
nominatim admin --check-database
```

Install Python dependencies for the Nominatim API to interact with the database

```bash
$USERHOME/nominatim-venv/bin/pip install psycopg[binary] falcon uvicorn gunicorn
$USERHOME/nominatim-venv/bin/pip install nominatim-api
```

Test the Nominatim installation by performing a search for Kathmandu

```bash
nominatim search --query Kathmandu
```

Switch to 'admin' user revoke superuser privileges from the 'nominatim' user

```bash
sudo -u postgres psql
ALTER USER nominatim WITH NOSUPERUSER;
\q
```

Create a systemd service to run Nominatim as a Gunicorn application

```bash
sudo tee /etc/systemd/system/nominatim.service << EOFNOMINATIMSYSTEMD
[Unit]
Description=Nominatim running as a Gunicorn application
After=network.target

[Service]
Type=simple
User=www-data
Group=www-data
WorkingDirectory=/srv/nominatim/nominatim-project
Environment="PATH=/srv/nominatim/nominatim-venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

ExecStart=/srv/nominatim/nominatim-venv/bin/gunicorn -b 0.0.0.0:8088 -w 4 -k uvicorn.workers.UvicornWorker nominatim_api.server.falcon.server:run_wsgi
ExecReload=/bin/kill -s HUP $MAINPID

StandardOutput=journal
StandardError=journal
PrivateTmp=true
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target
EOFNOMINATIMSYSTEMD
```

Reload systemd, enable and start the Nominatim service

```bash
sudo systemctl daemon-reload
sudo systemctl enable nominatim.service
sudo systemctl restart nominatim.service
```

#### Switch to the admin user and install Nginx, then configure it as a reverse proxy for Nominatim

Install Nginx web server

```bash
sudo apt-get install -y nginx
```

Configure Nginx to act as a reverse proxy for Nominatim by setting up a server block

```bash
sudo tee /etc/nginx/sites-available/osm.shreeniraula.com.np << EOF_NGINX_CONF
server {
listen 80; # Listen on port 80 for incoming HTTP requests
server_name osm.shreeniraula.com.np; # Specify the domain name for this server block

    location / {  # Define a location block to handle requests to the root URL
        proxy_pass http://127.0.0.1:8088;  # Forward requests to the Nominatim service running on localhost:8088
        proxy_set_header Host \$host;  # Pass the original host header to the proxied server
        proxy_set_header X-Real-IP \$remote_addr;  # Forward the real client IP address
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;  # Pass along the client’s IP address in the chain
        proxy_set_header X-Forwarded-Proto \$scheme;  # Pass along the protocol (HTTP or HTTPS) of the original request
    }

}
EOF_NGINX_CONF # Save the configuration to the Nginx sites-available directory
```

Remove the default symbolic link for the default Nginx configuration

```bash
sudo rm /etc/nginx/sites-enabled/default
```

Enable the website configuration by creating a symbolic link

```bash
sudo ln -s /etc/nginx/sites-available/osm.shreeniraula.com.np /etc/nginx/sites-enabled/
```

Test the Nginx configuration for any syntax errors

```bash
sudo nginx -t
```

Restart Nginx to apply the changes

```bash
sudo systemctl restart nginx
```

Fix permission issues for Nominatim by ensuring the correct ownership

```bash
ls -ld /run
sudo chown -R www-data:www-data /srv/nominatim
```

Verify the Gunicorn version for debugging purposes

```bash
sudo -u www-data /srv/nominatim/nominatim-venv/bin/gunicorn --version
```

Manually run Gunicorn for troubleshooting if the service does not work

```bash
/srv/nominatim/nominatim-venv/bin/gunicorn -b 0.0.0.0:8088 -w 4 -k uvicorn.workers.UvicornWorker nominatim_api.server.falcon.server:run_wsgi
```

Check the Nominatim service logs for any errors or troubleshooting

```bash
sudo journalctl -u nominatim.service
```

##

#### Test the functionality of the Nominatim Python frontend (API) by starting the server and performing a search query

```bash
# Manually start the Nominatim API server if it is not already running, to enable the frontend for search and geocoding requests
nominatim serve

# Test the search API with a query for Kathmandu and get the results in JSON format
curl http://127.0.0.1:8088/search?q=Kathmandu&format=json
```

##

#### Install Certbot and configure HTTPS for secure access to the site

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d osm.shreeniraula.com.np
```

##

#### Querying OpenStreetMap API for Location Details and Reverse Geocoding in Kathmandu

Query the OpenStreetMap API for the location 'Kathmandu' to get details in JSON format

```bash
curl https://osm.shreeniraula.com.np/search?q=Kathmandu&format=json
```

Perform reverse geocoding by passing latitude and longitude to retrieve the address in Chandragiri Municipality, Kathmandu

```bash
curl https://osm.shreeniraula.com.np/reverse?lat=27.6892881&lon=85.2321257&format=json
```

### Congratulations! You have successfully installed the Nominatim API with OpenStreetMap data.
