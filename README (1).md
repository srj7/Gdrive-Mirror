
# Dockerizing Laravel App



### What’s different in production?
#### Source files:
In development, we usually mount the current working directory into a container so that changes can be made to the source code and will be immediately reflected in the running application. In production however we’ll copy our source files directly into images so that they can be run independent of the host (ie: on a server somewhere).
#### Exposed ports:
In development we’ll sometimes bind ports on the host to containers to enable easier debugging. This is not required in production as services can communicate through the network docker-compose creates automatically for us.
#### Data persistence:
In development we setup MySql with a named volume to allow the data written to it to persist (even when our containers stop and restart) — but we didn’t handle situations where the app may want to write to disk (in the case of Laravel, this happens with views) — so, in production we’ll need to create a volume which will allow the data that’s written to disk to persist.
#### Environment Variables
In development we just had a .env file that was available to the container along with all the rest of our source code. In production we’ll load a different one in dynamically
#### SSL & Nginx config
We’ll need to configure our web server to enable secure connections, but we’ll want to load the path to the certs dynamically at run time so that we can swap live ones for self-signed when testing the production setup locally.
### Step 1 — Prepare the ‘app’ image
In the first post we created a PHP-FPM based image that was suitable for running a Laravel application. Then all we needed to do was mount our current directory into that container and the app worked. For production we need to think about it differently though, the steps are:
Use a Laravel-ready PHP image as a base

Copy our source files into the image (not including the vendor directory)

Download & Install composer

Use composer within the image to install our dependencies

Change the owner of certain directories that the application needs to write to

Run PHP artisan optimize to create the class map needed by the framework.
So let’s dive in. First up you’ll want to create the app.dockerfile that we’ll use to accomplish all of this.
```
Notes:
On line 1 we’re building from an image I’ve published to the Docker hub that contains just enough to run a Laravel Application.
Because on line 3 & 5 we use the COPY command, Docker will check the contents of the what’s copied each time we attempt a build. If nothing has changed, it can use a cached version of that layer — but if something does change, even a single byte in any file, the entire cache is discarded and all following commands will execute again. In our case that means every time we attempt to build this image, if our decencies have not changed (because the compose.lock file is identical) then Docker will not execute the RUN command that contains composer install and our builds will be nice and fast!
```
#### Step 2 — create .dockerignore
In the previous snippet we saw the line COPY . /var/www This alone would include every single file (including hidden directories like .git) resulting in a HUGE image. To combat this we can include a .dockerignore file in the project root.
It works just like the .gitignore you’re used to, and should include the following as a minimum.
.git
.idea
.env
node_modules
vendor
storage/framework/cache/**
storage/framework/sessions/**
storage/framework/views/**
Notes:
The last three lines are there to ensure any files written to disk by Laravel in development are not included, but we do need the directory structures to remain.
#### Step 3 — Build the ‘app’ image
With the files app.dockerfile and .dockerignore created, we can go ahead and build our custom image.
```
docker build -t reponame/app-name:tag .
```
Notes:
This may take a few minutes whilst the dependencies are downloaded by composer.
-t here instructs Docker to ‘tag’ this image. You can use whatever naming convention you like, but you need to consider where this image is going to end up when deciding. In this example I’m publishing this image under my username on docker hub (shakyshane) and want the repo to be named laravel-app You can see what I mean here https://hub.docker.com/r/shakyshane/laravel-app/ & for the more visual amongst us:

When the image has built successfully, you can run docker images to verify the image is tagged correctly.
Step 4— Add the ‘app’ service to docker-compose.prod.yml
Now we can begin to build up the file docker-compose.prod.yml starting with our app service. (as in the first post, these are all under the ‘services’ key, but don’t worry as there will be full snippets later).
#  The Application
app:
  image: shakyshane/laravel-app
  volumes:
    - /var/www/storage
  env_file: '.env.prod'
  environment:
    - "DB_HOST=database"
    - "REDIS_HOST=cache"
Notes:
We use image: shakyshane/laravel-app here to point to the image we just built in the last step. Remember this has all of our source code inside it, so we do not need to mount any directories from our host.
We need a way for files written to disk by the application to persist in the environment in which it runs. Using a volume definition in this manner /var/www/storage will cause Docker to create a persistent volume on the host that will survive any container stop/starts.
We’re going to set up Redis as the session and cache driver, but I’ve yet to find a way to stop Laravel writing view caching to disk, that’s why this single volume is required.
We use env_file: ‘.env.prod’ to mount a Laravel environment file into the container. This part is something that can be improved by using a dedicated secret-handling solution, but I’m not dev-opsy enough to do that, and in cases where we’re just using a single-server setup, I think this approach is ok. (please, any security experts out there, correct/point me in the right direction)
Step 5 — Prepare the ‘web’ image
So, now that we’re building a custom image that contains a PHP environment, all of our source code and all application dependencies, it’s time to do the same for the web server.
This one is much simpler. We’re just going to build Nginx, copy a vhost.conf into place (suitable for a Laravel app), and then copy in the entire public directory. This will allow nginx to serve our static files that do not require processing by the application (such as images, css, js etc)
So go ahead now and create web.dockerfile . It only needs the following:
FROM nginx:1.10-alpine

ADD vhost.conf /etc/nginx/conf.d/default.conf

COPY public /var/www/public
Notes:
This time we’re using nginx:1.10-alpine — the -alpine bit on the end means this base image was built from a teeny tiny linux base image that will shave 100s of MB from our final image size.
The alpine build also supports HTTP2 out of the box also, so, bonus.
Step 6 — Create the NGINX config
Save the following as vhost.conf — that’s the name we had above in web.dockerfile

Notes:
Where you see paths like/etc/letsencrypt/live/example.com in that file above, you can change example.com for your own domain if you have one, but either way, go and download some self-signed certs from http://www.selfsignedcertificate.com/ (or generate your own if you know how) and stick them in the following directory certs/live/example.com — it should look like this when done.

The idea is that when testing locally, you can pass in something like LE_DIR=certs — referring to your local directory, but then on the server you can pass LE_DIR=/etc/letsencrypt — which is where the certbot will dump your certs. This will create security warnings locally due the self-signed certificates, but you can click past the warning to fully test HTTP2 + all your links over a HTTPs connection.
Step 7 — Build the ‘web’ image
With the files web.dockerfile and vhost.conf created, we can go ahead and build our second custom image.
docker build -t shakyshane/laravel-web .
Notes:
Just as the previous image, we tag this one to match the repo name under which it will live later.
Step 8— Add the ‘web’ service to docker-compose.prod.yml
# The Web Server
web:
  image: shakyshane/laravel-web
  volumes:
    - "${LE_DIR}:/etc/letsencrypt"
  ports:
    - 80:80
    - 443:443
Notes:
We mount the directory containing our certificates using an environment variable. Docker-compose will replace ${LE_DIR} with the value we provide at run time which will allow us to swap between live/local certs.
We bind both port 80 & 443 from host->container. This is so that we can handle both insecure and secure traffic — you can see this in the second server block in the vhost.conf file above.
Step 9 — Add MySql and Redis services to docker-compose.prod.yml
Finally, the configuration for MySql and Redis needs to be placed in our docker-compose.prod.yml file — but for these we do not need to build any custom images.
version: '2'
services:

  # The Database
  database:
    image: mysql:5.6
    volumes:
      - dbdata:/var/lib/mysql
    environment:
      - "MYSQL_DATABASE=homestead"
      - "MYSQL_USER=homestead"
      - "MYSQL_PASSWORD=secret"
      - "MYSQL_ROOT_PASSWORD=secret"
  
  # redis
  cache:
    image: redis:3.0-alpine

volumes:
  dbdata:
Notes:
We use a named volume for MySql to ensure data will persist on the host, but for Redis we don’t need to do this as the image is configured to handle this for us.
Step 10 — Create .env.prod
As with any Laravel Application, you’re going to need a file containing your app’s secrets, a file that is usually different for each environment. Now because we want to run this application in ‘production’ mode on our local machine, we can just copy/paste the default Laravel .env.example sample file and rename to .env.prod — then when this application ends up on a server somewhere we can create the correct environment file and use that instead.
Step 11 — Test in production mode, on your machine!
This is where things start to get seriously cool. We’ve built our source & dependencies directly into images in a way that allows them to run on any host that has Docker installed, but the best bit is that this includes your local development machine!. Say goodbye to test locally, pushing and hoping for the best!
At this point we just have a final command to run, but it’s worth recapping what you should’ve done up to this point.
Created a app.dockerfile , built an image from it & configured it in docker.compose.prod.yml
^ same again for web.dockerfile
Created a .dockerignore file for exclude files and directories from COPY commands
Created the vhost.conf file with NGINX configuration, created self-signed certs for local testing & added the docker-compose.prod.yml config for it
Added the Redis & MySql configurations
Created a env.prod configuration file
Your docker-compose.prod.yml should look something like the following:
version: '2'
services:

  #  The Application
  app:
    image: shakyshane/laravel-app
    working_dir: /var/www
    volumes:
      - /var/www/storage
    env_file: '.env'
    environment:
      - "DB_HOST=database"
      - "REDIS_HOST=cache"

  # The Web Server
  web:
    image: shakyshane/laravel-web
    volumes:
      - "${LE_DIR}:/etc/letsencrypt"
    ports:
      - 80:80
      - 443:443

  # The Database
  database:
    image: mysql:5.6
    volumes:
      - dbdata:/var/lib/mysql
    environment:
      - "MYSQL_DATABASE=homestead"
      - "MYSQL_USER=homestead"
      - "MYSQL_PASSWORD=secret"
      - "MYSQL_ROOT_PASSWORD=secret"

  # redis
  cache:
    image: redis:3.0-alpine

volumes:
  dbdata:
Once that’s in place, all that’s left to do is run a single command…
LE_DIR=./certs docker-compose -f docker-compose.prod.yml up
And in a few moments your application will be accessible at https://0.0.0.0
Congratulations, you’re now running your application in a way which is 100% reproducible on a production server — no kidding! Once the 2 images we built are published to a registry like Docker Hub, all you’d need to do is place the docker-compose.prod.yml & an .env file on a server and your application will be running using the exact system you already tested on your local machine.
This is developer bliss.