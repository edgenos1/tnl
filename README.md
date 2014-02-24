# The Command Line Tunnel Manager
### Real Programmers Don't Need GUIs!

Sometimes, when you use `ssh`, you need to first log into a _gateway_ machine before you can log into the system you want:

    me@myhost:~$ ssh destsystem
    ssh: Could not resolve hostname destsystem: nodename nor servname provided, or not known
    me@myhost:~$ ssh gateway
    Last login: Thu Jan 30 15:47:50 2014 from 10.55.5.53
    me@gateway:~$ ssh destsystem
    Last login: Thu Jan 03 10:34:12 2014 from gateway.local
    
If you use `ssh`, you can do something called _tunneling_ that can allow you to log in directly from one your system directly to your destination system without first logging in each time to the _gateway_.

`ssh` does this by allowing you to create a tunnel on an unused port, and using `ssh` to use that port:

    $ ssh -N -L gateway -L 2001/destsystem/22
    
This creates a tunnel on Port `2001` to the system `destsystem` taking to `destsystem`'s `ssh` daemon which is listening onto Port 22 (the standard `sshd` port).

Now if I want to log into `destsystem`, I can do so directly from `myhost`:

    me@myhost:~$ ssh -p 2001 localhost
    me@gateway:~$ ssh destsystem
    Last login: Thu Jan 03 10:34:12 2014 from gateway.local
    
There are several software programs that allow you to manage these tunnels. However, they all seem to share a few issues:

* They can be a pain to setup. You usually have to define a gateway, then give all the various systems for that gateway, filling in port numbers, names, etc.
* Some systems make it very difficult to see the configuration. What port is my tunnel to my SQL DB machine connected to? You usually have to bring up the configuration dialog box over and over.
* Even once you've set up the tunnels, you still have to remember how you mapped the various machines to port numbers.
* Most tunnel handlers only concern themselves with the setup, but aren't very helpful with the connection.

I wanted a system of some intelligence. Where it will figure out the port mapping for me. Where I can tell it _I want to log into this system_ and it figures out how to do that. Where I copy, paste, and modify configurations. Where I can easily take the status of the gateways and tunnels.

Thus, I created this tunnel manager.

## Overview

To use the Command Line Tunnel Manger, you need to configure the gateways and systems via the `$HOME/.tnlrc` file. Once that is done, you can use `tnl` and it's various sub-commands to start and stop the gateway, log onto remote systems, and to query the status of the gateways and systems.

Once you have everything configured via the `$HOME/.tnlrc` file, you have to start up the tunnel on your gateway via the `tnl start` command:

    me@myhost:~$ tnl start gateway
    
As long as your gateway tunnel is up and running, you can use `tnl ssh` to log into your remote hosts:

    me@myhost:~$ tnl ssh destsystem
    
One you've finished, and no longer need the tunnel connection, you can shutdown the tunnel:

    me@myhost:~$ tnl stop gateway

## Setting Up the Tunnel Resource File

All configuration is done by the Tunnel Resource File located at `$HOME/.tnlrc`. Both the systems you want to connect to and the gateways you're tunneling through are configured with this file. Being a plain text file means it's easy to copy, paste, and edit lines. 

By having a plain text file, it becomes easy to duplcate configurations. An example of one appears below:

    #
    # Gateways
    #
    gateway: -name detroit -system gtwdt1 -start 2000 -background
    gateway: -name wasau -system gtwws3 -start 5000 -background
    #
    # Systems
    #
    system: -name maven -system 10.55.1.32 -gateway detroit
    system: -name build -system bldws32
    system: -name jenkins -user tomcat -system bldws32 -crypt 3des


### Gateways

Gateways are configured with lines that start with the string: `gateway:`. The word `gateway` must be all lowercase and must be followed by a colon.

After that, you can specify various parameters using the same parameter syntax used on the command line with parameters starting with one or two _dashes_ followed by their value (if they have one). Below are a few examples:

    gateway: -name detroit -system gwydt01 -start_port 2000
    
This gateway is called `detroit` which is really system `gwydt01`. To start up this gateway, you would use:

    $ tnl start detroit
    
#### Gateway Parameters

The following are valid parameters for the gateway configuration:

* `-system <system_or_ip_address>`  
(**Required**) This is the actual system name or its IP address.

* `-start <port_num>`  
(**Required**) This is the starting tunnel port number. In most tunnel managers, you must map each system to a port, then remember that mapping. This program handles all of that for you. The first mapped tunnel takes this port number. The next one takes the next available port after this one, and so on.  
<nbsp>  
It may seem that things are even more difficult. How do I log onto a system if I don't know the port map? Again, it doesn't matter. With `tnl`, you connect via system name, and let the program figure out what port that system is mapped to.

* `-name <name>`  
(**Optional**) The name you want to give the gateway. This is the way you will refer to the gateway. Think of it as an alias of the actual system name. If the name is not given, the name will be set to whatever is `-system`  .

* `-background`  
(**Optional**) Put the gateway process into the background as soon as `ssh` connects to the gateway. You can't use this parameter if you need to enter a password. Instead, you must leave the terminal active, and be careful not to accidentally kill the background process.  
<nbsp>  
However, if you have setup [ssh-keygen](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man1/ssh-keygen.1.html) and created a public/private key pair, you can connect to the tunnel without having to enter a password. Then, you can use this parameter which not only throws the gateway/tunnel process into the background, but also sets `nohup` to keep it running even if you delete that particular terminal session. It is highly recommended to setup `ssh-keygen`, and then to use this parameter.

* `-user <user>`  
(**Optional**) The name of the user to log into the gateway. By default, this will either be the current user, or the user configured in the [ssh configuration file](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man5/ssh_config.5.html#//apple_ref/doc/man/5/ssh_config). Setting this will override the default user and the user configured in the ssh configuration file.

* `-crypt <cipher_spec>`  
(**Optional**) This is the cipher method to use in encrypting the data between your local host and the gateway machine. In most `ssh`, the default is `3des` which is DES encryption used with three different keys. See your [ssh manpage](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man1/ssh.1.html) for more information.

* `-port <port_num>`  
(**Optional**) The port number used by the gateway's `sshd` daemon. Normally, this *should* be Port 22. However, certain companies block Port 22, and `sshd` has to operate over another port. By default, port 22 will be used.

* `-debug <num_between_0_to_3>`  
(**Optional**) The debug level used when connecting. Zero (the default) means no debugging information is sent to STDERR. 1 means basic debugging information is sent, and the higher this number, the more debugging information is send to STDERR.

### Systems

Systems are the systems you need to log into, but need to _tunnel_ through the gateway to get to the system.

Systems are configured in the same `$HOME/.tnlrc` file as the gateways. However, their lines begin with the word `system:` all in lower case and followed by a colon.

Like gateways, after the `system:` string, you specify  various parameters using the same syntax used on the command line with parameters starting with one or two dashes followed by their value if a value (if there is one). Below is an example:

    system: -name database -system db32dt01
    
The system name you use is `database`, but it connects to the system `db32dt01`. Once the gateway is started, you can log into `db32dt01` like this:

    $ tnl ssh database
    
#### System Parameters

The following are valid parameters for the system configuration:

* `-system <system_or_ip_address>`  
(**Required**) This is the actual system name or its IP address.

* `-tunnel <port_num>`  
(**Optional**) This is the port number you want to use for this particular system. Normally, you will let `tnl` figure out the ports to use since you don't have to know the ports to log into a particular system. However, if you want to force a mapping between a particular system and port, you can do it with this parameter.

* `-name <name>`  
(**Optional**) The name you want to give the gateway. This is the way you will refer to the gateway. Think of it as an alias of the actual system name. If the `-name` parameter is not passed, it will be the same as the `-system` name.

* -`gateway <gateway_name>`  
(**Optional**) Normally, when a gateway is started all systems not already with an active tunnel will be used by that gateway. However, there are times when you must match a particular system with a particular gateway. This parameter allows you to force an association between a gateway and a system.

* `-crypt <cipher_spec>`  
(**Optional**) This is the cipher method to use in encrypting the data between your local host and the gateway machine. In most `ssh`, the default is `3des` which is DES encryption used with three different keys. See your [ssh manpage](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man1/ssh.1.html) for more information.

* `-port <port_num>`  
(**Optional**) The port number used by the system's `sshd` daemon. Normally, this *should* be Port 22. However, certain companies block Port 22, and `sshd` has to operate over another port. By default, port 22 will be used.

* `-debug <num_between_0_to_3>`
(**Optional**) The debug level used when connecting. Zero (the default) means no debugging information is sent to STDERR. 1 means basic debugging information is sent, and the higher this number, the more debugging information is send to STDERR.

* `-x11`  
(**Optional**) Allow X11 protocol forwarding through this tunnel. This will allow X11 programs to display their GUI interface onto your local system.

* `-user <user>`  
(**Optional**) The name of the user to log into the system. By default, this will either be the current user, or the user configured in the [ssh configuration file](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man5/ssh_config.5.html#//apple_ref/doc/man/5/ssh_config). Setting this will override the default user and the user configured in the ssh configuration file.

***Note***: There is nothing stopping you from defining the same system with two different _alias_ names, and then using this parameter to force a login to a particular user. 

For example:

    system: -name build   -system bld01dt02
    system: -name jenkins -system bld01dt02 -user jenkins

You could log into system `bld01dt02` as either:

    $ tnl ssh build
    
which will log you in as the default user, or

    $ tnl ssh jenkins
    
which will log you in onto the same system, but this time as user _jenkins_.

### Comment lines

In the `$HOME/.tnlrc` file, lines that start with `#` are considered comments.

## Sub-Commands

`tnl` takes several sub-commands. Each command will start with `tnl`, then the sub-command, and finally a parameter if required.

    $ tnl start <gateway>
    
Starts the gateway `<gateway>`. This command will go through the various systems configured in the `$HOME/.tnlrc` file, and see which ones do not have an active tunnel. Then, will start up those systems with this particular gateway (unless the system configuration ties that system to another gateway).

The systems will use the `-start` port number, as the first port up to the number of ports required for the tunneling. If a port is already mapped, another port will be selected. If a system _hard mapped_ to a particular port, that port will be used if it is not already taken.

    $ tnl stop <gateway>
    
Stops the tunneling process on `<gateway>` all active connections will be lost, and you must restart the gateway to connect to the various systems on the gateway.

    $ tnl ssh jenkins
    
Connects to the system you called `jenkins` using the configuration for that system. Note that you do not have to know the port number being used for the tunnel.

    $ tnl ls
   
List the various systems as configured. The output looks like this:

    * Name       System          Port  User       Gateway  X11? Debug Crypt
    * jenkins    10.50.2.123     2001             Detroit  No          
      maven      mvnwsdc02                        Washou   No          
    * tj         10.50.2.123     2001  jenkins    Detroit  No         3des

If a line starts with an asterisk, the tunnel for that system is active and you may log into it. The _Port_ is either the port that is hard coded into the configuration, or is the active tunneling port. The gateway is either the hard coded gateway in the `$HOME/.tnlrc` file or the gateway which is acting as the tunnel for that particular system.

In the above, the tunnel for `tj` and `jenkins` are active while the tunnel for `maven` is not. Note that both `tj` and `jenkins` both use Port 2001. They actually both go to the system with the IP address of 10.50.2.123, but one will use the default user and one will use _jenkins_ as the user name.

    $ tnl lsgwy
    
Lists the gateways configured in the `$HOME/.tnlrc` file. The output looks like this:

    PID     Name             System           Start  Port   Crypt           Debug
            washou           gwyws01          2000   22     3des            
     24173  detroit          gwydr02          2000   22     3des

The PID lists the active process ID of that system, and if it has a number in that column, the gateway is currently running.


    $ tnl help
    
Gives you the basic synopsis of the various `tnl` sub-commands.
    
    $ tnl tnlrc
    
Gives you information on setting up the _Tunnel Resource File_.
    
    $ tnl man`
    
Prints out the Complete Command Line Tunnel Manager Manpage

## Dependencies

* The program needs Perl 5.12 or higher installed.

* This program depends upon the actual `ssh` command being in the `$PATH`. It uses the `ssh` command to do the actual connections. Future versions may use the Perl [Net::SSH](https://metacpan.org/pod/Net::SSH) in the future, but an attempt was made not to use optional modules.

* This program uses the `pgrep` and `ps` commands and expects their flags to follow either the [BSD](https://en.wikipedia.org/wiki/Berkeley_Software_Distribution) or the [GNU](https://en.wikipedia.org/wiki/GNU) command line parameters. These commands must also be present and in the user's `$PATH`. Future versions of this program may use

##Bugs (I mean _Features_)

One of the things that SSH uses is the `known_hosts` file to track what hosts are mapped to particular host keys. Because of this file, changing the port mapping can cause problems with this program. You may see a warning like this:

    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    @    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
    Someone could be eavesdropping on you right now (man-in-the-middle attack)!
    It is also possible that a host key has just been changed.
    The fingerprint for the RSA key sent by the remote host is
    xx:xx:xx:xx:xx...
    Please contact your system administrator.
    Add correct host key in $HOME/.ssh/known_hosts to get rid of this message.
    Offending RSA key in $HOME/.ssh/known_hosts:1
    RSA host key for [localhost]:2000 has changed and you have requested strict  checking.
    Host key verification failed.

This can be caused by the systems you're tunneling to being remapped to alternate ports. The known host maps IP addresses, system names, etc. to _system fingerprints_. This makes it hard for a _man in the middle_ attack where an intruder is pretending to be another system. When you use SSH, it checks known hosts against their finger print in the `$HOME/.ssh/known_hosts` file. If this is different, you will get this warning.

This can happen because tunneled hosts are not mapped to their IP address or host name, but by the port number used by the tunnel. You change the ports that a particular host goes to, and you'll get the above error.

For example, let's say that `foo01` is mapped to port `2000` and `foo02` is mapped to port `2001`. Your `known_hosts` file will look something like this:

    [localhost]:2000 <The fingerprint for foo01>
    [localhost]:2001 <The fingerprint for foo02>
    
Now, if these get remapped, so `foo01` is using port `2001` instead of `2000`, and you attempt to log into `foo01`, SSH will look at the second line and see that host `foo01` has a different finger print from the previous host that mapped to port `2000`.

Of course, the whole idea was to let this program handle the port mapping, so how do you know what ports will be used?

This program attempts to assign the ports in the order that the systems appear in the `.tnlrc` file. If you add new systems to the _end_ of the `.tnlrc`, you should be okay. If you delete systems from the file, or add new systems to the middle of the file, you can end up causing mapping confusion. If this happens, delete the line in the `known_hosts` file that is causing problems, and allow the `known_hosts` file to remap that system.

Or, you can use the `-port` parameter on the system lines to force a particular system to be on a particular port. This will always prevent port remapping. However, it does mean that you have to maintain your own tunnel mapping.

I will attempt to fix this in a later version of this program as soon as I can figure out a way around this problem.

## Author

David Weintraub david@weintraub.name (Yes, that is a valid email address).

## Copyright

Copyright &copy; 2014 by the author, David Weintraub. All rights reserved. This program is covered by the open source BMAB license.

The BMAB (Buy me a beer) license allows you to use all code for whatever reason you want with these three caveats:

1. If you make any modifications in the code, please consider sending them to me, so I can put them into my code.

2. Give me attribution and credit on this program.

3. If you're in town, buy me a beer. Or, a cup of coffee which is what I'd prefer. Or, if you're feeling really spendthrify, you can buy me lunch. I promise to eat with my mouth closed and to use a napkin instead of my sleeves.

