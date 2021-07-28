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
a few emails from users. If you want to debug it you'll need to ssh into
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
`EC2>Load Balancing>Target Groups>gru-backend-0-7-0>Targets` to track the
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
Mosaic instance. If it has issues, you need to ssh into mosaic.chpc.utah.edu.
Then su to the ucgd-peanalysis user (you might not have permissions to do this.
Ask Yi, Al, or Matt, and worst case Shawn Rynearson has access to this
command):

```bash
sudo su - ucgd-peanalysis
```

Now attach to my running ucgd-peanalysis tmux session:

```bash
tmux a
```

One window has a stealthcheck instance running in
`/scratch/ucgd/lustre/work/proj_UCGD/ucgd-peanalysis/stealthcheck/instance1`.
It will take care of keeping gru and Mosaic running.

If it crashes for some reason, you can restart it by going to that directory
and running:

```bash
../stealthcheck
```

If that doesn't work for some reason, you can run Mosaic and gru manually as
with the AWS services above. Just look in the `instance1/checks.json` file for
the proper commands to run.


## CDDRC

We have a relatively new instance of Mosaic running on the machine
`cddrc.utah.edu` for the CDDRC project. It shares the gru backend with the main
instance. It's also less critical than the main instance. Currently if it goes
down you're just going to have to wait for Anders to get back, since it's
running on his personal UNID account.


[0]: https://github.com/anderspitman/stealthcheck

[1]: https://github.com/iobio/iobio-stealthcheck/blob/master/stealthcheck1.iobio.io/checks.json

[2]: https://github.com/iobio/iobio-backend-services/blob/master/docs/fixing_phenolyzer.md
