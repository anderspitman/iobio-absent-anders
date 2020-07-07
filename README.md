# Backend

The backend deployment is relatively robust.


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


### fibridge

fibridge is used for local file support in the iobio apps. It should be kept
alive by stealthcheck. It's probably not the end of the world if it goes down
for a couple days, though we'll likely get a few emails from users. If you
want to debug it you'll need to ssh into ubuntu@lf-proxy.iobio.io and figure
out why the executable isn't running on port 9001. I've never had it stay down
since using stealthcheck.


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

Our PE deployment is currently a bit hacky, but also simple. It just consists
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

One window is running gru, the other is running Mosaic. To switch between them
do `CTRL-B n`. You can read up on tmux for more commands.


[0]: https://github.com/anderspitman/stealthcheck
