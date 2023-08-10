# Backend

The backend deployment is relatively robust. The stealthcheck service should
keep everything running. I've given a rundown of all our services to aid in
debugging in case something unexpected happens.


## General information


### SSH access

All of the machines involved should be accessible by the `ubuntu` user via the
`iobioServers.cer` private key. If you don't have a copy ask Yi or Anders.


### stealthcheck

Most of our services, both on AWS and CHPC, are controlled by stealthcheck.
[stealthcheck][0] is a very simple monitoring service I built. It does health
checks on our other services, and sends an email alert and attempts to restart
them if they go down.


## AWS

### stealthcheck

There are 2 instances running, stealthcheck1.iobio.io and
stealthcheck2.iobio.io. stealthcheck1 is the one that monitors all the other
services, including stealthcheck2. The only purpose of stealthcheck2 is to
monitor stealthcheck1.

See the private repo [here](https://github.com/iobio/iobio-stealthcheck) for
more information about stealthcheck, including how to set up our AWS
stealthcheck deployment from scratch if either of the VMs becomes
unrecoverable.

However, if you run into problems with stealthcheck, it's recommended to just
shut it down (can simply stop stealthcheck1 and stealthcheck2 from the AWS
console) and running the services manually until Anders gets back to fix it.
All of the necessary commands are in the [checks.json file][1] for
stealthcheck1 (look under the `fail_command` keys). You can SSH into the
various machines and run the commands shown. You'll lose monitoring and email
alerts (ie you'll need to keep an eye on things manually), but it should be
sufficient to limp along for several days.

The only two that are actually important are fibridge and
`services.backend.iobio.io` (which is currently just for phenolyzer). You can
pretty safely ignore the others. Information for each service is below.


### fibridge

fibridge is used for local file support in the iobio apps. It's probably not
the end of the world if it goes down for a couple days, though we'll likely get
a few emails from users.

First thing to try is rebooting (NOT terminating) the instance from the AWS
EC2 dashboard.

If you want to debug directly you'll need to ssh into
ubuntu@lf-proxy.iobio.io and figure out why the executable isn't running on
port 9001.

There are a couple unique things that can go wrong with fibridge. The first
is if the TLS cert expires. We're currently using a Let's Encrypt cert,
so we have to manage it manually. You can see from the stealthcheck config
where the cert is stored. In order to renew it, you need to run:

```
sudo certbot renew
```

But that won't work as long as fibridge is running on port 80, so you
need to kill fibridge first. But stealthcheck will restart it. You could
stop stealthcheck, but what I usually do is `sudo killall fibridge-rs`
and then quickly run `sudo certbot renew` before fibridge comes back
up.

**NOTE:** If you're running fibridge manually (ie stealthcheck is down), it
can appear to be working when it isn't. One way to verify it's actually ok
is by running bam.iobio with local files. If it doesn't work there's a good
chance fibridge needs to be restarted. Checking once a day is probably
sufficient.


### Phenolyzer

Phenolyzer runs on services.backend.iobio.io. This has actually been quite
temperamental lately, going down every few days. I haven't had time to track
down the issue. You'll need to keep an eye out for monitoring emails, or if
you're monitoring manually (ie stealthcheck is down), you can use the
following curl to quickly verify that it's running:

```
curl --fail --max-time 5 https://services.backend.iobio.io/
```

If that command fails, instructions for fixing phenolyzer are [here][2].


### quarantest

quarantest should be kept alive by stealthcheck. In any case it's a
non-critical service, so if it goes down just wait until Anders gets back.


### gru

The first thing to do if you suspect something is wrong with the AWS gru
backend is perform an instance refresh from the AWS console.

This is accomplished by going to `EC2>Auto Scaling>Auto Scaling
Groups>gru-backend>Instance refresh>Start instance refresh>Start`. I usually
change the settings to 50% and 60 seconds but it probably doesn't matter too
much.

It takes a number of minutes to complete. You can monitor the nodes under
`EC2>Load Balancing>Target Groups>gru-target-group>Targets` to track the
refresh progress.

If that doesn't solve the problem, you'll also need to make sure the DNS
(route 53), TLS certificates, and load balancer are working.

You can also try to SSH into the individual VMs (they are named like gru-1.X.X)
and see if you can find anything fishy going on. I would be very surprised if
it got to this point.


### Multialign

Tony is currently in charge of Multialign. She'll probably be the first to
notice if it breaks anyway. Eventually we'll get this managed by stealthcheck.


## CHPC

Our CHPC PE deployment is pretty simple. It just consists of gru instance and
Mosaic instance. These are both managed by
[systemd][3] services. The files for these
services are `/etc/systemd/system/iobio-gru-backend.service` and
`/etc/systemd/system/mosaic.service`. Inspecting those files will tell you
exactly how the services are run.

If something goes wrong, Chase would be the best person to ask to take a
look. You'll need sudo permissions for systemd in order to do most debugging.
`sudo -l` will tell you what commands you have access. Refer to the systemd docs
for how to use the commands `systemctl` and `journalctl`. If you don't have
access to the commands, you'll need to email `helpdesk@chpc.utah.edu`.


## CDDRC

We have an instance of Mosaic running on the machine
`cddrc.utah.edu` for the CDDRC project. It shares the gru backend with the main
PE instance.

The systemd file is at `/etc/systemd/system/mosaic.service`. Refer to the
CHPC instructions above for details.



[0]: https://github.com/anderspitman/stealthcheck

[1]: https://github.com/iobio/iobio-stealthcheck/blob/master/stealthcheck1.iobio.io/checks.json

[2]: https://github.com/iobio/iobio-backend-services/blob/master/docs/fixing_phenolyzer.md

[3]: https://en.wikipedia.org/wiki/Systemd
