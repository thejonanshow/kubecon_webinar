# Building a RaspberryPi Kubernetes Cluster
- Setup pis
- Install k3s
- Install OpenFaaS
- Deploy some functions
- Send Prometheus data to New Relic

These instructions assume you're using OSX. If you adapt them for another operating system please pull request your changes so future users will have a better time.

# Preparing your Raspberry Pis

Repeat these setup instructions for each SD card.

## Setup your SD cards

### Download raspios

Download the latest version of raspios from [raspberrypi.org](https://downloads.raspberrypi.org/raspios_lite_armhf_latest).

### Download balenaEtcher

We're going to use balenaEtcher to flash our SD cards. There's a direct download link for version `1.5.116` below but if you're reading this much after February of 2021 you may want to checkout the [latest releases](https://github.com/balena-io/etcher/releases/latest) to see if they've made any improvements.

Download [balenaEtcher (v1.5.116)](https://github.com/balena-io/etcher/releases/download/v1.5.116/balenaEtcher-1.5.116.dmg)

### Flash your SD cards

Select `Flash from file` and find the zip file for raspios that you downloaded above. It probably looks something like:
```
<some-date>-raspios-buster-armhf-lite.zip
```

You can flash your SD card directly from that zip file, there's no need to decompress it first.

![balenaEtcher 'Flash from file' screenshot](https://user-images.githubusercontent.com/270746/108016712-0ac8b480-6fc8-11eb-8672-95c50b69e0e4.png)

Now choose `Select target` and check the box next to your SD card, you should be able to tell which one it is by the size but double check by ejecting the card and putting it back if you're not sure. You don't want to overwrite the wrong drive.

If you happen to own multiple SD card readers note that you can burn all of your cards at once in this step by just selecting each of their boxes.

When you've selected your card(s) click on `Flash!` to flash them, it shouldn't take longer than a couple of minutes.

## Enable SSH

If you flashed multiple cards remove them all and insert them one at a time for this next step. If you only have one inserted you'll need to remove it and reinsert it here anyway so you can see the filesystem.

When a Raspberry Pi first boots it will automatically enable SSH if it finds a file named `ssh` in the `/boot` directory.

Create an empty file on `/boot` with the `touch` command so your pi will boot with ssh enabled:
```
touch /Volumes/boot/ssh
```

### Eject the disk

You can eject the disk in Finder by clicking the eject icon next to `boot`. If you'd like to use the command line instead you'll need your disk name. Find your disk name with:
```
diskutil list
```

The disk you're looking for will have a `Windows_FAT_32` partition named `boot`. Your disk name will look something like `/dev/disk5` but the number at the end will likely be different.

Now you can eject the disk with `diskutil` like this:

```
diskutil eject disk5
```

## Boot your pi

Insert your SD card into your pi and power it up. If you're having problems with power try a different USB-C cable, the cheaper the better. There was a flaw with the first batch of pis that prevents higher quality cables from working.

Make sure you boot your pis one at a time, the instructions below depend on being able to access your pi at `raspberrypi.local` which will only work if you only have one pi booted at a time.

When you first boot your pi a small script will run to expand your filesystem to the size of your SD card and then your pi will reboot once.

You'll know this process has completed when the green light stops blinking and you only see the red one (the green light indicates SD card activity).

## Copy your public key

We're going to use `ssh-copy-id` to copy your public key to the Raspberry Pi so we can login without a password.
The `StrictHostKeyChecking` and `UserKnownHostsFile` options aren't really necessary but they'll prevent this command from throwing errors if you run it on another pi in the future.

```
ssh-copy-id -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null pi@raspberrypi.local
```

When you're prompted for a password enter the default password for the pi user:
```
raspberry
```

## Connect to your pi

You should be able to SSH in to `raspberrypi.local` like this:
```
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null pi@raspberrypi.local
```

If you're prompted for a password again enter the password for your SSH key, not the pi user password we entered above.

## Setup control groups
Kubernetes uses control groups ([cgroups](https://en.wikipedia.org/wiki/Cgroups)) to control resource usage.

To enable cgroups on raspios include these kernel options at the end of `/boot/cmdline.txt`:
```
cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
```

You'll need root to edit that file so use `sudoedit`:
```
sudoedit /boot/cmdline.txt
```

Be careful not to include a newline at the end of the file, these options should be added at the end of the existing line with only a space separating them.

## Configure your pi

Raspberry Pi's include a helpful configuration utility. We're going to use it to change 3 things:
- The pi user's password
- The hostname
- The amount of memory that the GPU gets to use

To start the configuration utility run:
```
sudo raspi-config
```

### Change the password

The `Change password` option should be visible from the main menu. Even though you're just experimenting there are a lot of ways to inadvertently expose your pi to the internet, so it's a good idea to use a strong password.

### Change the hostname

This is how you're going to connect to your pi in the future, it will be used in place of `raspberrypi` in our SSH commands up above.

Go to `Network Options` and select `Hostname`, it will give you some of the rules you need to use for hostnames on the first screen so read carefully if you're new to this part.

Make sure you use a unique hostname for each of the pis in your cluster, we don't want two trying to show up on the network with the same name.

You may also consider giving your primary node a unique name so it's easy to remember. In my case I have 4 pis and I'm using these hostnames:
```
piqueen
pidrone0
pidrone1
pidrone2
```

### Decrease GPU memory

By default Raspberry Pis will dedicate rather a lot of memory to the GPU so you can run a display. Since we're not going to use a display (we're running 'headless') we want to lower the amount of memory for the GPU so Kubernetes gets more of it.

Go to `Advanced Options` and select `Memory Split`, then enter 16.

From the main menu you can hit escape to exit or select `Finish`. Select `Yes` when it asks you if you'd like to reboot.

We're going to run the rest of these commands on your computer so you can finish your SSH session with `exit`.

# Install Kubernetes

We're going to use Arkade to install Kubernetes. Technically we're going to be using a lightweight distribution called `k3s`, not full-blown Kubernetes, but since it conforms to the Kubernetes API you won't likely notice the difference.

## Download and install Arkade

[Arkade](https://github.com/alexellis/arkade) makes installing Kubernetes tools much easier, much like the Homebrew project.

There's a bash script available to install and configure Arkade, you can download it to a file called `ark.sh` like this:

```
curl -sLSo ark.sh https://dl.get-arkade.dev
```

Whenever you download bash scripts you should have a quick look through before you run them. Do whatever amount of inspecting on `ark.sh` seems reasonable to you and then run it with:
```
sudo bash ark.sh
```

## Install k3sup

The [k3sup](https://github.com/alexellis/k3sup) tool can be used to bootstrap a k3s cluster on any machine where you have SSH access.

Download `k3sup` using Arkade:
```
ark get k3sup
```

You'll be given some options in the post-install instructions to help you get started but I'd recommend you go with the path outlined below.

Arkade stores all of the tools you download with it in `$HOME/.arkade/bin/`. You could copy them to `/usr/local/bin` so you can run them but to keep things tidy I'd suggest you just add that to your path like this:
```
export PATH=$PATH:$HOME/.arkade/bin/
```

Remember that will only work for the duration of this terminal session so if you'd like to keep the change around you should add it to your shell configuration (`~/.zshrc` if you're using ZSH, the OSX default shell).

## Install k3s on your primary node

The install command for k3sup uses the IP address of your pi. Choose one of the hostnames you setup to use as your primary node and get the IP address with `ping`.

If you followed my example above we want to use `piqueen` in this case so:

```
ping piqueen.local
```

Copy the IP address and paste it into the k3s install command below. My primary node is on `192.168.1.6` so I'd run this:

```
k3sup install --ip 192.168.1.6 --user pi
```

You'll see rather a lot of commands being run on your behalf and when it's all setup you should have a new `kubeconfig` file in your local directory.

You can safely ignore the post-install instructions for now.

## Install kubectl

Most everything you do with your k3s cluster is going to use a tool called `kubectl`, which stands for "Kubernetes Control". Some people who hate cuddling will insist that you refer to this tool as "Kube Control" but the correct pronunciation is "Kube Cuddle", because it's for hugging your new cluster.

You can install kubectl with Arkade:

```
ark get kubectl
```

In order for kubectl to know which cluster you're trying to work with you need to tell it where your kubeconfig file lives by setting the `KUBECONFIG` environment variable.

You're hopefully still in the same directory where k3sup created that kubeconfig file for you so you can set that environment variable like this:

```
export KUBECONFIG=`pwd`/kubeconfig
```

You probably don't want to include this export command in your shell configuration because you might want to use different kubeconfig files to control other clusters in the future.

## Cuddle your kube

You can confirm that everything is working correctly at this point by telling kubectl to list all of the nodes:

```
kubectl get node
```

You should see something like this:

```
NAME      STATUS   ROLES    AGE   VERSION
piqueen   Ready    master   3m   v1.19.7+k3s1
```

## Install k3s on your secondary nodes

We're going to use k3sup again to install k3s on each of the secondary nodes in your cluster. This time we're joining an existing cluster so the command will change slightly.

In my case my remaining pis have the IP addresses `192.168.1.7`, `192.168.1.8`, and `192.168.1.9`, so my 3 commands look like this:

```
k3sup join --ip 192.168.1.7 --server-ip 192.168.1.6 --user pi
k3sup join --ip 192.168.1.8 --server-ip 192.168.1.6 --user pi
k3sup join --ip 192.168.1.9 --server-ip 192.168.1.6 --user pi
```

# Deploy OpenFaaS

First install faas-cli so we can interact with [OpenFaaS](https://github.com/openfaas):

```
arkade get faas-cli
```

Now install OpenFaaS itself:

```
arkade install openfaas
```

It will take a moment for everything to start up, you can check the status of the deployment like this:

```
kubectl get deployments -n openfaas
```

In order to use faas-cli we'll first need to authenticate so we need to grab our password with kubectl:

```
PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode; echo)
```

Now export your OpenFaaS URL as an environment variable (replace the hostname as necessary):

```
export OPENFAAS_URL=http://piqueen.local:31112
```

Authenticate with faas-cli using the password we saved earlier:

```
echo -n $PASSWORD | faas-cli login --username admin --password-stdin
```

Now you can list all of the built-in functions in the faas-cli store like this:

```
faas-cli store list --platform armhf
```

Note that we're filtering for ARM processors, not all of the functions will run on a Raspberry Pi.

## Deploying a function

We're going to start by deploying Figlet, an ASCII art generator:

```
faas-cli store deploy figlet
```

You can test your new function with curl:

```
curl http://piqueen.local:31112/function/figlet -d 'Hello World!'
```

If everything went smoothly you should see this:

```
 _   _      _ _        __        __         _     _ _
| | | | ___| | | ___   \ \      / /__  _ __| | __| | |
| |_| |/ _ \ | |/ _ \   \ \ /\ / / _ \| '__| |/ _` | |
|  _  |  __/ | | (_) |   \ V  V / (_) | |  | | (_| |_|
|_| |_|\___|_|_|\___/     \_/\_/ \___/|_|  |_|\__,_(_)
```

# Getting data into New Relic

OpenFaaS includes Prometheus so you're already collecting metrics on your new function. To get them into New Relic we're going to use the `remote_write` features of Prometheus.

[Sign up for a New Relic account](https://newrelic.com/signup?utm_source=devrel&utm_medium=kubeconwebinar&utm_campaign=DevRel) if you don't already have one.

Log in to your New Relic account and then visit the [Prometheus integration page](https://one.newrelic.com/launcher/nr1-core.settings?pane=eyJuZXJkbGV0SWQiOiJwcm9tZXRoZXVzLXJlbW90ZS13cml0ZS1pbnRlZ3JhdGlvbi1uZXJkbGV0cy5zZXR1cC1wcm9tZXRoZXVzIn0=&platform[timeRange][duration]=1800000&platform[$isFallbackTimeRange]=true).

Follow the instructions on the page to generate your `remote_write` configuration for `prometheus.yml`. It should look something like this:

```
remote_write:
- url: https://metric-api.newrelic.com/prometheus/v1/write?prometheus_server=openfaas
  bearer_token: <your-token>
```

Now edit the configmap that OpenFaaS set up for Prometheus:

```
kubectl edit configmap prometheus-config -n openfaas
```

You want to include that snippet just below the global section at the same indentation level:

```
22   prometheus.yml: |
23     global:
24       scrape_interval:     15s
25       evaluation_interval: 15s
26       external_labels:
27           monitor: 'faas-monitor'
28
29     remote_write:
30     - url: https://metric-api.newrelic.com/prometheus/v1/write?prometheus_server=openfaas
31       bearer_token: <your-token>
```

Now restart your Prometheus deployment so the new configuration takes effect:

```
kubectl rollout restart deployment/prometheus -n openfaas
```

Go back to New Relic and click on `See your data`.

As soon as Prometheus finishes restarting and scrapes the first data (it scrapes on 15s intervals) you will start to see data appearing in New Relic.

Go explore your data and build some dashboards!
