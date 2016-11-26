# Watch US Netflix from the UK using `ssh`
### When nested procrastination produces something useful

_DISCLAIMER: This has been written for educational purposes of learning about net
traffic routing. You should probably only employ this if you live in the US._

_NOTE: Yes, this all can be achieved through a run off-the-mill VPN. I like the 
detached nature of this solution from any other service. It will never ask you
to pay for more features. All parts of the solution are owned by you. It does not
bottleneck your speed and ask you to pay for higher priority.  As far as I can
tell, this runs at native speed._

This basic guide will outline the steps to set up a `SOCKS5` proxy over `ssh` on
an Amazon Web Services instance in the US to forward net traffic from a machine
with a foreign IP address to the host machine. I set this up for use with
Netflix, but presumably it can be used for other web services that are location
based.

The service is free for a year provided you don't run any other AWS instances on
the account that is used to set up the AWS instance. The AWS 'free tier' provides
you with 750 hours a month (where `31 * 24 = 744`) of time on a linux machine for
12 months, after which it is `$0.013` an hour. Or you can just set up another AWS
account, but presumably in a year this method won't work anymore.

The basic steps are outlined below
 - Sign up for an AWS account and launch a box in the US (Assumed `Ubuntu 16.04`)
 - Set up an account with a DDNS
 - On the AWS instance:
   - Set the configuration for the DDNS client
   - Install `docker`
   - Run a docker image of the `noip` client
 - On the host machine:
   - Add an `ssh` alias
   - Add alias to:
     - Set up a background `ssh` tunnel using the instance as a proxy
     - Run a browser which connects to the web through the proxy
     - Kill background `ssh` tunnel

At this point the set up works, and you can check your ip address if from the US
by googling 'what's my IP'. You can then add Amazon cloud watch scripts to,
for example, only run the AWS instance at specific times of day, where the DDNS
and SSH alias will handle the instance's dynamic IP.

## Account Set Up

#### AWS

We use AWS to spoof an IP address from a different location. AWS have servers in
the US, hence we use this service to set up a virtual machine on their hardware,
in the US. This means that any service sending web traffic to them, send it to a
US address, and so sends them traffic appropriate for the US. We can then read
the data from the virtual machine (we 'forward' the traffic to the UK).

 - Sign up to AWS
 - Follow the wizard to launch a free EC2 instance in some US region
   - AWS is fairly generous with marking out exactly which options are free
 - Generate a `.pem` key to set the instance up with and download it
   - If you're familiar with ssh keys, feel free to continue on to the next
   section
 - Generate an ssh keypair on your host
   - A short guide to do this is [here](http://www.linuxproblem.org/art_9.html),
   but there are many resources available
 - `ssh` into the AWS instance using the public IP address of the machine, found
 by clicking the instance in the AWS console and finding 'Public IP' in the details
   - The `ssh` command syntax is `ssh -i <pem-key> user@host`
   - The default amazon user is `ubuntu`
   - Hence an example command is `ssh -i ~/Downloads/key.pem ubuntu@54.192.170.23`
 - Place your normal public key in `~/.ssh/authorized_keys` on the instance if you
 have one, of continue to use the `.pem`
   - Using an authorized key skips manually routing to the `.pem` to login


#### DDNS

A DNS (domain naming service) is how text URLs such as `www.google.com`, are
translated to IP addresses, such as `74.125.236.195` - where the IP address is
what the network uses to point to a machine. A DDNS (Dynamic Domain Naming Service)
is useful if our IP address changes often, and we don't have to type in an
arbitrary sequence of numbers every time we wish to point to our machine. The
service updates the link between the text URL and IP address.

Why does our IP change? We have the cheapest machine possible on AWS and so our
physical location on their servers is the lowest priority.  If we shutdown our
instance and someone else needs the space, our instance will move to a different
IP address, hence we have to go through the hassle of finding the new IP address.
In general, each time we start our AWS instance, the IP address is not guarenteed
to be the same, hence the DDNS gives us a static character string to reference,
when pointing to our machine.

 - Signing up to a DDNS is as easy as signing up to any service. The recommended
   service, `noip`, is found [here](https://www.noip.com/sign-up)
   - At some point, you will have to put the password to the service in a plain
   text file.  If, like me, even though you know the file can be fully protected
   you do not like the idea of seeing your an important password in plain text,
   choose a different one.

## On the Remote Instance

Essentially, all we're doing here is setting up a client to ping the remote
instance's IP address up to the DDNS. How does this work? The client lives on
the remote instance, and so can find its own IP address when and if it changes.
We also give the client login details to our `noip` account.  Every 30 minutes,
we ask the client up to update our `noip` account with the IP address of the
machine it lives on.  Therefore, when we ask the DDNS who lives at
`my-url.ddns.net`, it's up to date to within the periodic polling time.

#### Docker

Docker is a service that runs other services in 'containers'. The added level of
complexity eases set up, as developers can very easily curate the environment
that their application will be set up in, minimising errors in simple applications.

 - `ssh` into the remote machine using the public IP address from the AWS console
 - Install docker using walkthrough [here](https://docs.docker.com/engine/installation/linux/ubuntulinux/)
   - If you've not used linux before, these docs are fairly accessible to follow. The set up amount to smashing `ctrl-c`, `ctrl-v` multiple times to achieve something fairly complex.

#### `noip`

 - Run this command to set up the docker image

```
docker run \
   -v /home/ubuntu/no_ip_config:/config \
   -v /etc/localtime:/etc/localtime \
   -d  --restart=unless-stopped \
   coppit/no-ip
```

 - The `noip` client will be downloaded and set running in its own sand-boxed
 environment.
 - Set up the `noip` configuration with the details you signed up to the service
 with by copy and pasting the below code into the terminal

```
# set up config on aws instance - use your own details ofc
mkdir ~/no_ip_config
echo "USERNAME='yupimaki" | tee ~/no_ip_config/noip.conf
echo "PASSWORD='ILoveDDNS'" | tee -a ~/no_ip_config/noip.conf
echo "DOMAINS='yupimaki.ddns.net'" | tee -a ~/no_ip_config/noip.conf
# Examples: 5 m, 5 h, 5 d. Minimum is 5 minutes.
echo "INTERVAL='30m'" | tee -a ~/no_ip_config/noip.conf
```

## On the Host (assuming a Ubuntu host)

We add an `ssh` alias for ease of using `ssh`. Instead of the length command that
was necessary previously, we after adding an alias we can log on to our remote
machine through `ssh`, simply by typing `ssh <alias>`.  Aliases are added by
adding new hosts and configuration to the `~/.ssh/config` file. Copy and paste
the below block into the host terminal, with the URL you chose for your remote
machine via `noip`.

```
# Add an ssh alias on host
echo "Host US-Jumpbox" >> ~/.ssh/config
echo "  Hostname yupimaki.ddns.net" >> ~/.ssh/config
echo "  User ubuntu" >> ~/.ssh/config
```
This will set up an alias such that `ssh US-Jumpbox` routes to
`ssh ubuntu@yupimaki.ddns.net`, to `ssh ubuntu@XX.XXX.XXX.XX`, where the IP
address will have been recently updated by the docker container running on the
remote instance.  If you didn't add a public key, an wish to keep using the `.pem`,
add an extra line to `~/.ssh/config` with `IdentityFile path/to/key.pem`.

## Final Aliases on Host

Finally, we need an easy way to set up our ssh tunnel to the proxy server and
launch a browser which is configured to connect to the web through the proxy.

I find the easiest way (by no means the correct way) to do this is to add some
lines to my `~/.bashrc` file.  

```
# Easy way to kill ssh tunnel
port-kill () {
   # Find offending process's port
   {
      # Truthy hack to handle error if no process with that name is found
      ps -C $1 -o pid= &> /dev/null && port=$(ps -C $1 -o pid=)
   } || {
      echo "No process found with name $1"
      return
   }

   # Get process name by port ID
   process_name=$(ps -p ${port} -o comm=)

   # Option to kill found process
   read -r -p "Are you sure you want to kill ${process_name}? [y/N] " response
   response=${response,,}    # tolower
   if [[ $response =~ ^(yes|y)$ ]]
   then
      kill ${port}
   fi
}

## Run and kill commands
# Must not have a google-chrome session open.
# Seems to be a bug in chrome that you can't force start with a new session
alias netflix-us="ssh -D 1089 -f -C -q -N US-Jumpbox && google-chrome --proxy-server='socks5://localhost:1089'"
alias kill-ssh="port-kill ssh"
```

Breaking down the final aliases, `netflix-us` is composed of two statements.

```
ssh -D 1089 -f -C -q -N US-Jumpbox
```

We already know what `ssh US-Jumpbox` does. Explaining the other flags;
 - `-D 1089` sets up [D]ynamic port forwarding on the local port `1089` through
 `ssh`
 - `-f` [f]orks the process and runs it in the background
 - `-C` [C]ompresses the data before sending it through the `ssh` pipe
 - `-q` sends all `STDOUT` and `STDERR` messages to `/dev/null`, i.e. it is
 [q]uiet

```
google-chrome --proxy-server='socks5://localhost:1089'"
```

This command simply starts google chrome, however specifies that it connects to
the web through a proxy server.  The server is our `localhost` at port `1089`,
which we know is dynamically routing our web traffic through the US.

Thanks for reading,

Akhil

### References
 - Route traffic using SOCKS tunnel
   - https://www.digitalocean.com/community/tutorials/how-to-route-web-traffic-securely-without-a-vpn-using-a-socks-tunnel
