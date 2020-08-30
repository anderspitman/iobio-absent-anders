# Backend

The backend deployment is relatively robust. The stealthcheck service should
keep everything running. I've given a rundown of all our services to aid in
debugging in case something unexpected happens.


## AWS


### stealthcheck

[stealthcheck][0] is a very simple monitoring service I built. It does health
checks on our other services, and sends an email alert and attempts to restart
them if they go down. It's been working quite well the past couple months.

There are 2 instances running, one on dev.backend.iobio.io (in the stealthcheck
directory), and one on lf-proxy.iobio.io. Look at
`/home/ubuntu/stealthcheck/checks.json` on dev.backend.iobio.io to see all the
commands used to check and run the various services. That should be very
helpful in debugging them if something goes wrong.

If something catastrophic happens and you can't access the `checks.json` files,
you can look at the following repos for backups:

* https://github.com/iobio/stealthcheck_config_1

* https://github.com/iobio/stealthcheck_config_2


### fibridge

fibridge is used for local file support in the iobio apps. It should be kept
alive by stealthcheck. It's probably not the end of the world if it goes down
for a couple days, though we'll likely get a few emails from users. If you
want to debug it you'll need to ssh into ubuntu@lf-proxy.iobio.io and figure
out why the executable isn't running on port 9001. I've never had it stay down
since using stealthcheck.

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

Another thing to look out for is if the fibridge machine reboots, port
80 needs to route to port 9001 and 443 to 9002. This is to avoid
running fibridge as root. You can easy google how to do this with 
the iptables command. There's also a rules.v4 file in the home directory
that you can use.


### quarantest

quarantest should be kept alive by stealthcheck. In any case it's a
non-critical service, so if it goes down just wait until Anders gets back.


### gru

Most problems with the AWS gru can be fixed by identifying naughty nodes and
terminating them. The auto scaling will take care of replacing it. You can look
at the monitoring information in the AWS console (the nodes are named like
`gru-backend-worker-0.XX.0`). If any of the nodes are consistently using more
than 10% CPU, go ahead and kill them. It probably means gru has a renegade
process spinning its wheels. It's not a bad idea to check on this every few
days.

If that doesn't solve the problem, you'll also need to make sure the DNS
(route 53), TLS certificates, and load balancer are working.


### Phenolyzer

Phenolyzer runs on services.backend.iobio.io.


### Multialign

Tony is currently in charge of Multialign. She'll probably be the first to
notice if it breaks anyway. Eventually we'll get this managed by stealthcheck.


## PE

Our PE deployment is pretty simple. It just consists
of gru instance and Mosaic instance. If it has issues, you need to ssh into
mosaic.chpc.utah.edu. Then su to the ucgd-peanalysis user (you might not have
permissions to do this. Ask Yi, Al, or Matt, and worst case Shawn Rynearson has
access to this command):

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

[0]: https://github.com/anderspitman/stealthcheck
