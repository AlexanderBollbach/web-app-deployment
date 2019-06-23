# How to deploy multiple web apps on ec2
----
### Background
You have a number of web apps (demos, experiements, personal portfolio, etc) that you'd like others to see.  You looked into several hosting services but found them to be too expensive.  I will overview a (quasi) systematic approach to deploying dozens of apps for less than $10/month.

### Ingredients
- running ec2 instance (t2.micro works fine for small scale)
- a 'repos' dir where you push all the apps 'bare' repos to
- git 'post-update' hooks for triggering building
- an 'apps' dir where build contents live
- pm2 for running multiple node servers indefinitely
- Nginx (reverse proxy) for forwarding public URL's to individual apps
- a DNS provider for your domain name (optional)

### Creating the 'App':
An 'App' is just a certain folder structure that will contain all the requirements for deployment to ec2 with my apprach.  The structure looks as follows.  I will go into detail about what each file is for.
```
/your-app
    /client
        package.json
        webpack.config.json
        /src
            index.js
        /dist
            index.html
            main.js
    /server
        server.js
        package.json
    .gitignore
```

`/client` could be a complex React app using webpack, babel and whatever so long as it finally builds its output to `/dist` dir with an index.html and main.js.  (this is the normal webpack/npm workflow)  

`/server` will contain another `npm` project which is your server.  Again, in some of the more dynamic apps it will contain routes, run a graphQL server, or call cloud databases, but the simplest version just needs to build an `Express` app that servers up `../dist/index.html`.

A basic snippet for that server would look like this:
```
var express = require("express");
var app = express();
const path = require("path");
const dir = path.resolve(__dirname + "/../client/dist/");
app.use(express.static(dir));
app.listen(process.env.PORT || 8090);
```

> TODO: the 8090 hard-coded port here is bad.  I currently need to hard code different ports for all my apps such that they don't clash.  I am thinking of how to automate this process (possibly using environment variables from pm2/nginx/git-hook/shell-scripts).

### Make your local git repo

We will use `git` to push this project to the ec2.  assuming you've created your app according to this project structure, now create the git repo:

```
git init
git add .
git commit -m "first commit"
```

Your app is now a git repo and you can iterate on it locally and add new commits.  We're now ready to configure the ec2 to deploy whenever you want by pyshing to it.

### Configuring your ec2 for git triggered deployments

What we are going to do is use "bare repos" and "hooks" to achieve a automated build/deployment.  A bare repo is one that only contains branches and commits but no working directory.  The working directory will be checked out elsewhere into your `apps` directory where it will be booted up as a running node process.

First, lets connect to our ec2:
```
ssh -i YourPrivateKey.ssh
// in ec2..
home/ec2-user/:
```

#### step 1:
create your `/repos` dir if it doesn't already exist in the `/home` dir, and create your bare repo.
```
mkdir repos
cd repos
mkdir your-app
cd your-app
git init --bare
```
#### step 2:

switch back to your local machine and add your newly created ec2 bare repo as a remote:

```
git remote add origin ec2-255-255-255-255.compute-1.amazonaws.com:/home/ec2-user/repos/your-app.git
```

#### step 3:

write your `post-update` shell script.  This is what really handles our automated deployment.  The way this works is in your remote bare repo there is a `/hooks` dir that contains a bunch of hook scripts.  The `post-update.sample` will fire after you've pushed commits.  But we need to add the shell scripting code and rename the file to `post-update` (without the .sample) so it will actually run.

`post-update` (.sh)

```
#!/bin/sh

GIT_WORK_TREE=/home/ec2-user/apps/your-app git checkout -f
cd /home/ec2-user/apps/your-app

cd client
npm run build

cd ..
cd server
npm run build

pm2 stop "your-app"
pm2 start server.js --name "your-app"
```

After every push of new commits this script will navigate to the repo run the npm build scripts in the client and server dirs and run the node process with `pme2`.

#### step 4:

Finally, we'll need to update our nginx config file to forward from a URL like `yourDomain.com/your-app` to the pm2 node server.  There is a lot to nginx but this config works for me.  What I do for each app is to add a new `location` directive that routes to the port I gave that particular app.  

> TODO: Here again there is room for improvement.  I am thinking of how I could have nginx dynamically discover new apps by lookg at `/home/ec2-user/apps/` and start forwarding / privision the ports automatically.  

```
worker_processes auto;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    log_format main '"$request" $status';
   
    server {
    access_log /home/ec2-user/log main;
    listen 80;
    server_name localhost;

    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $remote_addr;

    location / {
        proxy_pass http://localhost:8080;
    }

    location /some-app/ {
        rewrite /some-app/(.*) /$1 break;
        proxy_pass http://localhost:8080;
    }

    # NEWLY ADDED
    location /your-app/ {
        rewrite /your-app/(.*) /$1 break;
        proxy_pass http://localhost:8081;
    }
  }
}

```

### How does this all work?

With all the previous setup for `your-app` this is how it gets deployed.  When you write some new code and commit it, you'll push to ec2 `git push origin master`.  it will be received by `yourEc2:/home/ec2-user/repos/your-app.git` and the `post-update` shell script will build the client and server into `/apps/your-app`.  It will also run the node server with `pm2` on a new port.  When you go to `yourDomain.com/your-app` the nginx server will forward `/your-app/` to `localhost:8081` (or some other port) where the pm2/node process is running.

Although the configuration is quite complicated you'll be able to automatically deploy new code with a single line `git push`.












