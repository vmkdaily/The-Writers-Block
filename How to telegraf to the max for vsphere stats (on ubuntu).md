
#### How to telegraf to the max for vsphere stats (on ubuntu)

## Introduction
In this write-up, we get Ubuntu Linux up and running with InfluxDB, Telegraf and Grafana to collect and visualize VMware vSphere performance data.

## About the TICK stack
The `influxdata` repository gives us access to the entire `TICK` stack (`Telegraf`, `InfluxDB`, `Cronograf`, and `Kapacitor`). We will only use `Telegraf` and `InfluxDB` of that stack, but we also show installing `Cronograf` too since that gives us great insight into localhost (more on that later). The `k` of the stack is for `Kapacitor` (not used), which can monitor and alert; We will skip that one but check it out!

## VMware KB
Configure VC for proper stat collection before proceeding.

```
  https://kb.vmware.com/s/article/2107096
```

## System Info
The system used for this is Ubuntu 16.04 Linux (Ubuntu 18.x is fine too) with 4 vCPU and 8GB RAM;

Also this system has a second `.vmdk` consumed as a /data drive.

You can optionally go with only 2 vCPU for smaller deployments. No additional disks are required, though may be desired for high performance deployments.

If running on VMware vSphere, ideally you should add an additional VMware Paravirtual SCSI controller (i.e. SCSI 1:0) for your data drive. If going extreme high performance, `meta` would be SCSI 2:0 and so on.

Finally, use a vmxnet3 network adapter for best performance.

## Credits
```
  https://twitter.com/jorgedlcruz  #dashboard author
  https://github.com/prydin        #plugin author for vsphere telegraf
```
## vsphere telegraf plugin project page
```
https://github.com/influxdata/telegraf/tree/release-1.8/plugins/inputs/vsphere
```
## Prerequisites
Before starting, create a read-only user in `VMware vCenter Server`. This will be used by`telegraf` to collect performance stats. Later we will enter this value in plain text into the `telegraf` configuration file.

> Warning: This solution uses plain text (see above). There are options you can pursue to be more secure such as using the vCenter Certificate and https for all communication. In this write-up, we do all clear text and http.

1. Using a web browser, login as `administrator@vsphere.local` to your vCenter server. Do not use the HTML5 client such as `/ui`), but instead use the full featured web client (i.e. with Adobe Flash). 

2. Navigate to `Menu > Administration > Single Sign On > Users and groups` (or similar).

3. Click the `users and groups` (or similar) in the left pane and in the right pane, click the drop-down to change from `localos` to `vsphere.local` and click `Add User`.

4. Name the user as desired, enter a password, and click `Add`.

5. In the left pane, navigate to `Menu > Access Control > Global Permissions`, and in the right-pane, add a read-only permission for your user and check the box to propogate. Click ok.

6. Log out of vSphere client, or if desired, take a snapshot of the VM you will be working on and then log out.

## Getting started
Begin by logging in (i.e. via SSH) to the node that willl run InfluxDB, etc. You will need `root` or a user with `sudo`. After we create the user we will switch to that user and then use `sudo` for the remainder of the article. Let's get started.

## Optional - add a user
This is not required, but we can add a user to manage influxdb. We will just name the user `influxdb`.
```
  sudo adduser influxdb
```
## Optional - add user to sudo
```
  sudo usermod -aG sudo influxdb
```
## Optional - check if PowerShell is installed
We do not need PowerShell for this deployment at all, but in case you want it, we can first check if it is installed.
```
  pwsh
```
## Optional - Install PowerShell
Here, we install using the `snap` technique, which is the way to go. Remember, adding PowerShell is optional and not used for this write-up, other than optionally installing it now.
```
  sudo snap install powershell --classic
```

> Note: If you are on Ubuntu Desktop you can also install visual studio code as a snap with `sudo snap install code --classic`

## Exit and/or login as desired user before proceeding
Login now as `influxdb` if you created that or any other desired user. Next, we will install InfluxDB and other required components.

## Add the `influxdata` repository
We begin by adding the influxdata official repo; this gives us access to the `TICK` stack.

```
  curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -
  source /etc/lsb-release
  echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
```


> Note: If the above looks confusing, there are 3 lines. There is the `curl` line, then the `source` line, and finally the `echo` line. You can just paste it all at once.

## Install InfluxDB
```
  sudo apt -y update
  sudo apt -y install influxdb
```
## Show the binaries
Optionally, use the `which` command to show the location of the binaries. These are automatically in your path so no worries either way.
```
  which influx
  which influxd
```
## Edit the InfluxDB config file
```
  sudo nano /etc/influxdb/influxdb.conf
```

## Optional - If using data drives for InfluxDB
By default, InfluxDB will create everything it needs in the default locations.  However, we can optionally use our own folders that we create. My setup has a `/dev/sdb` (device does not matter) that is mounted as `/data`. Below, we just create some friendly folder names for each of the data locations that InfluxDB uses.

```
  cd /data  #cd to your extra drive if any
  sudo mkdir meta
  sudo mkdir influx_data
  sudo mkdir wal
```

## Optional - Assign permissions with `chown`
If you created a user, you can optionally assign permissions. Here, we assign permissions to the user we created previously called `influxdb`. This is not required and the InfluxDB defaults are fine.
```
  sudo chown influxdb:influxdb /usr/bin/influx*
```

> Note: Optionally, also do the above action on your target data folders, if manually created.

## Start InfluxDB
Finally, we are ready to start influxdb. 
```
  sudo systemctl start influxdb
```

> Note: Many of our steps taken were optional, and one can typically just install influxdb, start it up and get to using it.

## Get InfluxDB status
We've done a lot so far; let's check the status.
```
  sudo systemctl status influxdb
```

## Optional - Restart influxdb
This not required, but if you make a config change you can always restart with the following:
```
  sudo systemctl restart influxdb
```

## Install telegraf
Next, we install `telegraf` which is the app that collects all the datapoints for us. This is part of the influxdata `TICK` stack.
```
  sudo apt-get install telegraf
```

## About the telegraf config file
The telegraf app has a config file that comes fully loaded with all apps available, but has them commented out by default (expected).

> Warning: If you were born and raised on PowerShell, the way they comment these config files is going to feel like a nightmare. Get yourself a zen audiobook and play that as needed.

## Configure telegraf
We can use `vi` or `nano` to edit our selection.
```
  sudo nano /etc/telegraf/telegraf.conf
```

> Reference the telegraf vsphere github for details about the config file at https://github.com/influxdata/telegraf/blob/release-1.8/plugins/inputs/vsphere/README.md

## Optional - About Extra configuration files for telegraf
This trick comes from our dashboard author https://twitter.com/jorgedlcruz; The beauty of this one is that we can choose not to edit the default config file at all; Instead, we can craft the file containing only the elements we want.

## Extra config file location
Optionally, create or copy the desired config file to `/etc/telegraf/telegraf.d/<your file>.conf`. You can create an empty file with `vi`, `touch` or similar, but we just use `nano`. Change your filename as desired.
```
  sudo nano  /etc/telegraf/telegraf.d/vsphere-stats.conf
```

> Note: Using the extra config location is optional.

## Start telegraf
Now we are ready to start the telegraf service. This will read in the vcenter name and login we provided to the configuration file previously.
```
  systemctl start telegraf
```

## Optional - Install cronograf
The `cronograf` app can visualize like Grafana but will not be our primary in this write-up.  However, it is good for looking at the stats of your system that is running collections, etc.
```
  sudo apt-get install chronograf
```

## Optional - Start cronograf
```
  sudo systemctl start chronograf
```

## Optional - Login to cronograf
Open a web browser to your server by fqdn or using localhost. The cronograf app listens on port `8888` by default.
```
  http://localhost:8888
```

> Tip: Find the monitor section to review your system stats. Observe that you can get additional stats for your system network interface by enabling a plugin (similar to the vsphere one we did earlier); The config file you would need to edit is `/etc/telegraf/telegraf.conf`, then restart the service with `systemctl restart telegraf`. Now, you are getting some excellent detail into `cronograf`.

## More About cronograf
This is relatively new, and is a competitor to grafana. Because it is part of the TICK stack it would seem to be the easy choice; However, because grafana has been around for a bit, the charts we find are still using grafana and not cronograf (yet!).

> I came here to drink milk, and get stats; and I've just finished my milk.

## Install Grafana
Here, we install Grafana which is mandatory for the charts we will ultimately use.
```
  sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
  curl https://packages.grafana.com/gpg.key | sudo apt-key add -
  sudo apt -y update
  sudo apt -y install grafana
```

## Set Grafana to start automatically
```
  sudo systemctl enable grafana-server.service
  sudo systemctl daemon-reload
```

## Start Grafana
```
  sudo systemctl start grafana-server
```

## Show Grafana status
```
  sudo systemctl status grafana-server
```

## Install pie-chart plugin
```
  sudo grafana-cli plugins install grafana-piechart-panel
```

## Restart Grafana
```
  sudo systemctl restart grafana-server
```

## About grafana login `http` vs `https`
By default, your grafana installation will serve up an `http` page for login. Not shown here, but you can configure for `https` if desired. See grafana.com for more details.

## Login to grafana
Login with default of `admin` `admin`. You can optionally, click on `skip` when it prompts you to change it.
```
  http://yourserver:3000
```

> The default port for Grafana is `3000`. You can optionally change that in the grafana config file, typically at `/etc/grafana/grafana.ini`. To determine your config file location, `sudo systemctl status grafana-server`.

## Manual Steps from client - Install graphs
You'll want the following chart IDs which can be downloaded right from your Grafana instance. Use a browser to login to your deployment, and then click `Import`; Next, simply enter each of the following Chart IDs to add these excellent community vSphere charts:
```
  8159,8162,8165,8168
```

## Favorite the Charts
Within your deployment, navigate to each Grafana chart that you added and click the favorite icon. The favorite system is internal only for your deployment. By clicking favorite, it keeps the main charts on your home page Grafana.

-end-

