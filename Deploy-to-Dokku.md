# Deploy your S3Proxy to Dokku

[Dokku](http://dokku.viewdocs.io/dokku/) is a Docker powered open source Platform as a Service that runs on any hardware or cloud provider.

## Prerequisites
Setup a [Dokku](http://dokku.viewdocs.io/dokku/installation/#installing-via-other-methods) environment in your cloud provider of choice. For example, you can follow this [guide](http://dokku.viewdocs.io/dokku/getting-started/install/azure/) to install a Dokku instance on Microsoft Azure.

## Creating a new Docker App for S3Proxy

- From your local dev machine, git clone this repo:
```
$ git clone https://github.com/ritazh/s3proxydocker.git
```
- SSH into the Dokku vm in your Dokku instance
```
$ ssh -i ~/.ssh/dokku [[dokkuAdminCreatedDuringInstallation]]@[[dnsNameForPublicIP]].[[location]].cloudapp.azure.com
```
- Create a new Dokku app, let's call this app `s3proxy`:
```
$ dokku apps:create s3proxy
```

## Updating Your S3Proxy Dockerfile
Similar to running S3Proxy as a container with Docker, refer to the [Getting Started](https://github.com/ritazh/s3proxydocker#getting-started) section in the guide to update relevant properties in your [s3proxy.conf](s3proxy.conf). Specifically, for the `s3proxy.virtual-host` property, replace [[dokku app name for s3proxy]] with the new Dokku app you created in the previous step and replace [[dokku public ip]] with the new Dokku instance you have previously provisioned. 

For example, if I have the following in my `s3proxy.conf`:
```
s3proxy.virtual-host=http://s3proxy.123.456.789.1.io/
```
Then `s3proxy` is the dokku app name for my s3proxy app and `123.456.789.1` is the public ip of my dokku instance.

You can see an example of this in [s3proxydokku](s3proxydokku).

## Pushing Your S3Proxy App
- From your local dev machine, add a Dokku remote to your local repo using the Dokku username and push the app:
```
$ ssh-add <your-dokku-deploy-private-key>
$ git remote add dokku dokku@[[DNSNAMEFORPUBLICIP]].[[LOCATION]].cloudapp.azure.com:[[app name]]
$ git push dokku master
```
## Updating Your S3Proxy App
- When you need to update the solution, simply follow the usual git flow to push updates to dokku:

```
$ git add
$ git commit 
$ git push dokku master
```

## Configuring Your Subdomain for S3Proxy App
- Since S3Proxy treats all buckets and containers as sub-subdomains, we need to add a wildcard for our sub-subdomains to your Dokku app subdomain. You can do one of the following:

[Option 1] update your app's nginx.conf manually on the Dokku VM:

```
$ sudo vi /home/dokku/s3proxy/nginx.conf
$ sudo nginx -s reload
```

[Option 2] Update the nginx domain server name with a wildcard sub-subdomain url, use the Dokku domains plugin.

From the Dokku vm:
```
$ dokku domains:add s3proxy *.s3proxy.[[PublicIP]].xip.io
```
Sample output:

```
-----> Configuring s3proxy.[[PublicIP]].xip.io...
-----> Configuring *.s3proxy.[[PublicIP]].xip.io...
-----> Creating http nginx.conf
-----> Running nginx-pre-reload
       Reloading nginx
-----> Configuring s3proxy.[[PublicIP]].xip.io...
-----> Configuring *.s3proxy.[[PublicIP]].xip.io...
-----> Creating http nginx.conf
-----> Running nginx-pre-reload
       Reloading nginx
-----> Added *.s3proxy.[[PublicIP]].xip.io to s3proxy
```
To review the latest domains for the S3Proxy Dokku app:
```
$ dokku domains s3proxy
```
Sample output:

```
=====> s3proxy Domain Names
s3proxy.[[PublicIP]].xip.io
*.s3proxy.[[PublicIP]].xip.io
```

To review the updated nginx.conf, do the following:
```
$ sudo cat /home/dokku/s3proxy/nginx.conf 
```
Sample output:
```
server {
  listen      [::]:80;
  listen      80;
  server_name s3proxy.[[PublicIP]].xip.io *.s3proxy.[[PublicIP]].xip.io ;
  access_log  /var/log/nginx/s3proxy-access.log;
  error_log   /var/log/nginx/s3proxy-error.log;

...
```
## Increasing Client Request Maximum Size for Upload

To include additional configurations of nginx, we can add new configurations files under `/home/dokku/myapp/nginx.conf.d/`. 

```
$ sudo su
$ mkdir /home/dokku/myapp/nginx.conf.d/
$ echo 'client_max_body_size 50M;' > /home/dokku/myapp/nginx.conf.d/upload.conf
$ chown dokku:dokku /home/dokku/myapp/nginx.conf.d/upload.conf
$ nginx -s reload
```
## Scaling Your S3Proxy App
Increase your app web process to 2 containers.

```
$ dokku ps:scale s3proxy web=2
-----> Scaling s3proxy:web to 2
-----> Releasing s3proxy (dokku/s3proxy:latest)...
-----> Deploying s3proxy (dokku/s3proxy:latest)...
-----> DOKKU_SCALE file found
=====> web=2
[omit for brevity]
-----> Waiting for 10 seconds ...
-----> Default container check successful!
=====> s3proxy container output:
[omit for brevity]
=====> end s3proxy container output
[omit for brevity]
-----> Waiting for 10 seconds ...
-----> Default container check successful!
=====> s3proxy container output:
[omit for brevity]
=====> end s3proxy container output
-----> Running post-deploy
-----> Configuring s3proxy.[[Dokku public ip]].xip.io...
-----> Configuring *.s3proxy.[[Dokku public ip]].xip.io...
-----> Creating http nginx.conf
-----> Running nginx-pre-reload
       Reloading nginx
-----> Setting config vars
       DOKKU_APP_RESTORE: 1
-----> Shutting down old containers in 60 seconds
=====> 3322dad2a114953a9bd99c03b90f3e680f571c5c1bbd5e43a5255de25e95ae77
=====> Application deployed:
       http://s3proxy.[[Dokku public ip]].xip.io
       http://*.s3proxy.[[Dokku public ip]].xip.io
```

When looking at running processes for S3Proxy app, notice now we have 2 containers running:

```
$ dokku ps s3proxy
-----> running processes in container: 673aa4ae40305a3ddae8ac73239106337cd869b81a60781da9684ce702a41b26
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
[omit for brevity]
-----> running processes in container: 51471577d1e6081b18c3b16e1d93944656455843d34c422c68d64fe2dbde306d
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
[omit for brevity]
```