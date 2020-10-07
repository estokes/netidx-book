# A Small Example

Suppose we have a small daemon that we run on many computers on our
network, and it knows many things about them, and does many things. I
won't specifiy exactly what it does or everything it knows because
that's irrelevant to the example. However suppose one of the things it
knows is the current CPU temperature of the machine it's running on,
and we would like access to that data. We heard about this new netidx
thing, and we'd like to try it out on this small and not very
important case, what code do we need to add to our daemon, and what
options do we have for using the data?

We can modify our Cargo.toml to include netidx, and then add a small
self contained module, publisher.rs

``` rust
use anyhow::Result;
use netidx::{
    config::Config,
    path::Path,
    publisher::{Publisher, Val, Value},
    resolver::Auth,
};

#[derive(Clone)]
pub struct HwPub {
    publisher: Publisher,
    cpu_temp: Val,
}

impl HwPub {
    pub async fn new(host: &str, current: f64) -> Result<HwPub> {
        // load the site cluster config from the path in the
        // environment variable NETIDX_CFG, or from
        // dirs::config_dir()/netidx.json if the environment variable
        // isn't specified, or from ~/.netidx.json if the previous
        // file isn't present. Note this uses the cross platform dirs
        // library, so yes, it does something reasonable on windows.
        let cfg = Config::load_default()?;

        // for this small service we don't need authentication
        let auth = Auth::Anonymous;

        // listen on any unique address matching 192.168.0.0/16. If
        // our network was large and complex we might need to make
        // this a passed in configuration option, but lets assume it's
        // simple.
        let publisher = Publisher::new(cfg, auth, "192.168.0.0/24".parse()?).await?;

        // We're publishing stats about hardware here, so lets put it
        // in /hw/hostname/cpu-temp, that way we keep everything nice
        // and organized.
        let path = Path::from(format!("/hw/{}/cpu-temp", host));
        let cpu_temp = publisher.publish(path, Value::F64(current))?;
        Ok(HwPub {
            publisher,
            cpu_temp,
        })
    }

    pub async fn update(&self, current: f64) -> Result<()> {
        // update the current cpu-temp
        self.cpu_temp.update(Value::F64(current));

        // flush the updated values out to subscribers
        self.publisher.flush(None).await
    }
}
```

Now all we would need to do is create a HwPub on startup, and call
HwPub::update whenever we learn about a new cpu temperature value. Of
course we also need to deploy a resolver server, and distribute a
cluster config to each machine that needs one, that will be covered in
the administration section.

## Using the Data We Just Published

So now that we have our data in netidx, what are our options for
consuming it? The first option, and often a very good one for a lot of
applications is the shell. The netidx command line tools are designed
to make this interaction easy, here's an example of how we might use
the data.

``` bash
#! /bin/bash

netidx subscriber $(netidx resolver list /hw/ | grep 'cpu-temp$') | \
while IFS='|' read path typ temp; do
    IFS='/' read -a pparts <<< "$path"
    if ((temp > 75)); then
        echo "host: ${pparts[2]} cpu tmp is too high: ${temp}"
    fi
done
```

Of course we can hook any logic we want into this, the shell is a very
powerful tool after all. For example one thing we might want do is
modify this script slightly, filter the entries with cpu temps that
are too high, and then publish the temperature and the timestamp when
it was observed.

``` bash
#! /bin/bash

netidx subscriber $(netidx resolver list /hw/ | grep 'cpu-temp$') | \
while IFS='|' read path typ temp; do
    IFS='/' read -a pparts <<< "$path"
    if ((temp > 75)); then
        echo "/hw/${pparts[2]}/overtemp-ts|string|$(date)"
        echo "/hw/${pparts[2]}/overtemp/temp|f64|$temp"
    fi
done | \
netidx publisher --bind 192.168.0.0/24
```

Now we've done something very interesting, we took some data out of
netidx, did a computation on it, and published the result into the
same namespace. Our /hw namespace now has additional useful data in it
for each host. We can now subscribe to e.g. /hw/krusty/overtemp-ts and
we will know when that machine last went over temperature. To a user
looking at this namespace in the browser (more on that later) there is
no indication that the over temp data comes from a separate process,
on a separate machine, written by a separate person. It all just fits
together seamlessly as if it was one application.

Now I've made this seem great, but there are actually to problems
here. The first is in fact exactly the same sentence as the last
paragraph. There is no indication anywhere that this overtemp data is
a dirty bash script possibly running on a developer workstation that
is about to get switched off to install patches. That is why netidx
has access controls on publishing, and that's why it's a good idea to
use them even if your data is not sensitive. If you don't, anyone can
publish anything anywhere.

The second problem is more serious, in that, the above code will not
do quite what you might want it to do. You might, for example, want to
write the following additional script.

``` bash
#! /bin/bash

netidx subscriber $(netidx resolver list /hw/ | grep 'overtemp-ts$') | \
while IFS='|' read path typ temp; do
    IFS='/' read -a pparts <<< "$path"
    ring-very-loud-alarm ${pparts[2]}
done
```

To ring a very loud alarm when an over temp event is detected. This
would, in fact, work as expected, it just would not be as timely as
you might want. The reason is that the subscriber practices linear
backoff when it's instructed to subscribe to a path that doesn't
exist. This is a good practice, in general it reduces the cost of
mistakes on the entire system, but in this case it could result in
getting the alarm hours, or longer after you should. The good news is
there is a simple solution, we need to publish all the paths from the
start, but fill them will null until the event actually happens. That
way the subscription will be successful right away, and the alarm will
sound immediatly after the event is detected. So lets change the code ...

``` bash
#! /bin/bash

cat <(
    netidx resolver list /hw | \
        while IFS='/' read -a pparts
        do
            echo "/hw/${pparts[2]}/overtemp-ts|string|null"
            echo "/hw/${pparts[2]}/overtemp|string|null"
        done
) \
<(
   netidx subscriber $(netidx resolver list /hw/ | grep 'cpu-temp$') | \
       while IFS='|' read path typ temp
       do
            IFS='/' read -a pparts <<< "$path"
            if ((temp > 75)); then
                echo "/hw/${pparts[2]}/overtemp-ts|string|$(date)"
                echo "/hw/${pparts[2]}/overtemp/temp|f64|$temp"
            fi
       done
) | netidx publisher --bind 192.168.0.0/24

```

So first we list all the machines in /hw and publish null for
overtemp-ts and overtemp for each one, and then using cat and the
magic of process substitution we append to that the real time list of
actual over temp events.

## Or Maybe Shell is Not Your Jam

It's entirely possible that the above solution gives you night sweats,
or maybe your boss has an adversion to shell scripts, however hot you
think your creation is. In that case it's perfectly possible (and
probably more readable) do do the above in rust.

``` rust

```