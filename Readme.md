

# Get up and running with .Net Core and Ubuntu or other linux

This document is based on [Scott Hanselman](https://twitter.com/shanselman) [post](https://www.hanselman.com/blog/PublishingAnASPNETCoreWebsiteToACheapLinuxVMHost.aspx) 

## Step 0: Get a Linode and deploy Ubuntu LTS

## Step 1: Setup a sudo user

```shell
adduser mgl
usermod -aG sudo mgl
```

## Step 2: Install .Net Core depending on ubuntu distro 

### Step 2.1 Add the dotnet apt-get feed

 **Ubuntu 14.04 / Linux Mint 17**

```shell
sudo sh -c 'echo "deb [arch=amd64] https://apt-mo.trafficmanager.net/repos/dotnet-release/ trusty main" > /etc/apt/sources.list.d/dotnetdev.list' 
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 417A0893
sudo apt-get update
```
**Ubuntu 16.04 / Linux Mint 18**

```Shell
sudo sh -c 'echo "deb [arch=amd64] https://apt-mo.trafficmanager.net/repos/dotnet-release/ xenial main" > /etc/apt/sources.list.d/dotnetdev.list'
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 417A0893
sudo apt-get update

```

**Ubuntu 16.10**

```Shell
sudo sh -c 'echo "deb [arch=amd64] https://apt-mo.trafficmanager.net/repos/dotnet-release/ yakkety main" > /etc/apt/sources.list.d/dotnetdev.list'
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 417A0893
sudo apt-get update


```

### Step 2.2 Instal .Net Core SDK.

Before you start, please remove any previous versions of .NET Core from your system by using [this script](https://github.com/dotnet/cli/blob/rel/1.0.0/scripts/obtain/uninstall/dotnet-uninstall-debian-packages.sh). 

To install .NET Core 1.1 on Ubuntu or Linux Mint, simply use apt-get.

.NET Core 1.1 is the latest version. For long term support versions and additional downloads check the [all Linux downloads](https://www.microsoft.com/net/download/linux) section.

```Shell
    sudo apt-get install dotnet-dev-1.0.4
```

### Step 2.3 (optional):

Initialize some code

```Shell
    dotnet new console -o hwapp
    cd hwapp
```

Run the code

```Shell
dotnet restore
dotnet run
```

**NOTE: **If "dotnet restore" fails with a segmentation fault, you may be running into [this issue](https://github.com/dotnet/cli/issues/3681) with some 64-bit Linux Kernels. Here's [commands to fix it that worked for me on Ubuntu 14.04 when I hit this](https://github.com/dotnet/cli/issues/3681#issuecomment-242897595). The fix has been released as a NuGet now but it will be included with the next minor release of .NET Core, but if you ever [need to manually update the CoreCLR you can](https://github.com/dotnet/core/blob/master/release-notes/1.0/manual-shared-update.md).

## Step 3: Make an ASP .Net Core website

```Shell
mkdir dotnettest
cd dotnettest
dotnet new mvc
```

Now we can restore dependendencies and run our website 

```Shell
dotnet restore
dotnet run
mgl@ubuntu:~/dotnettest$ dotnet run 
Project dotnettest (.NETCoreApp,Version=v1.4) was previously compiled. Skipping compilation.
info: Microsoft.Extensions.DependencyInjection.DataProtectionServices[0]
      User profile is available. Using '/home/mgl/.aspnet/DataProtection-Keys' as key repository; keys will not be encrypted at rest.
Hosting environment: Production
Content root path: /home/mgl/dotnettest
Now listening on: http://localhost:5000
```

## Step 4: Expose the web to the outside

Although Hanselman speaks of exposing Kestrel to 80 port I have got an error due to permision and use of privileged ports by kestrel. As he states here

> You might have permission issues doing this and need to elevate the  dotnet process and webserver which is also a problem so let's just keep  it at a high internal port and *reverse proxy* the traffic with 
> something like Nginx or Apache. We'll pull out the hard-coded port from the code and change the Program.cs to use a .json config file. 

Modify your Program.cs and do not forget to import adecuated library for the configuration builder object

```c#
use Microsoft.Extensions.Configuration
```

```c#
public static void Main(string[] args)
{
    var config = new ConfigurationBuilder()
        .SetBasePath(Directory.GetCurrentDirectory())
        .AddJsonFile("hosting.json", optional: true)
        .Build();
 
    var host = new WebHostBuilder()
        .UseKestrel()
        .UseConfiguration(config)
        .UseContentRoot(Directory.GetCurrentDirectory())
        .UseStartup<Startup>()
        .Build();
 
    host.Run();
}
```

create the hosting.json file on the project root folder

```json
{
  "server.urls": "http://localhost:5123"
}
```

## Step 5 - Setup a Reverse Proxy like Nginx

I'm following the detailed instructions at the ASP.NET Core Docs site called "[Publish to a Linux Production Environment](https://docs.asp.net/en/latest/publishing/linuxproduction.html)." (All the [docs are on GitHub](https://github.com/aspnet/Docs/blob/master/aspnet/publishing/linuxproduction.rst) as well)

```shell
sudo apt-get install nginx
sudo service nginx start
```

Change the default  nginx site as follow.

```shell
sudo nano /etc/nginx/site-available/default
```

and change the content 

```json
server {
    listen 80;
    location / {
        proxy_pass http://localhost:5123;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```shell
sudo nginx -t 
sudo nginx -s reload
```

At this point you can check that your website is up and running you can open in your browser http://ip-address-of-your-linode.



## Step 6 : Keep your website running with supervisor

Supervisor is an app thet monitor your app and in case that fail restart it automatically, that's awesome.

Scott Hasselman effectively publish and copy published site to /var/dotnettest. This is perfect because ngnix has no permissions issues on this directory running under unpriviledged user as www-data. Said this, this is the moment where  you, me everibody has to be allert on cortrect paths on your server to move and configure supervisor without problems.

```Shell
dotnet publish
  publish: Published to /home/mgl/dotnettest/bin/Debug/netcoreapp1.0/publish
sudo cp -a /home/mgl/dotnettest/bin/Debug/netcoreapp1.0/publish /var/dotnettest
```

After that you have to make a configuration file for supervisor to maintain your website up and running

```shell
sudo nano /etc/supervisor/conf.d/dotnettest.conf
```

put this content inside 

```
[program:dotnettest]
command=/usr/bin/dotnet /var/dotnettest/dotnettest.dll --server.urls:http://*:5123
directory=/var/dotnettest/
autostart=true
autorestart=true
stderr_logfile=/var/log/dotnettest.err.log
stdout_logfile=/var/log/dotnettest.out.log
environment=ASPNETCORE_ENVIRONMENT=Production
user=www-data
stopsignal=INT
```

Stop and start supervisor , after that check supervisor log

```shell
sudo service supervisor stop
sudo service supervisor start
sudo tail -f /var/log/supervisor/supervisord.log
#and the application logs if you like
sudo tail -f /var/log/dotnettest.out.log 
```

## Step 7 : Todo the perfect deployment with Git 

//TODO