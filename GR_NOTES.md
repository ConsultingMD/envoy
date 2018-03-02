Tue Feb 27 17:49:30 PST 2018
Prior to trying any mods, used https://www.envoyproxy.io/docs/envoy/latest/install/sandboxes/local_docker_build to get me a local build world.

Build time is 500s then "out of space" to the action log. Ran out of space in my /tmp , which is configured to 10% of my RAM:

```
Filesystem      Size  Used Avail Use% Mounted on
udev            7.7G  4.0K  7.7G   1% /dev
tmpfs           1.6G  1.5M  1.6G   1% /run
/dev/sda1       470G  208G  238G  47% /
none            4.0K     0  4.0K   0% /sys/fs/cgroup
tmpfs           1.6G  9.5M  1.6G   1% /tmp
none            5.0M     0  5.0M   0% /run/lock
none            7.7G  144M  7.6G   2% /run/shm
none            100M   36K  100M   1% /run/user
```

Upped that to 50% of my ram:
sudo mount -o remount,size=50% /tmp

so now

```
udev            7.7G  4.0K  7.7G   1% /dev
tmpfs           1.6G  1.5M  1.6G   1% /run
/dev/sda1       470G  208G  238G  47% /
none            4.0K     0  4.0K   0% /sys/fs/cgroup
tmpfs           7.7G  9.5M  7.7G   1% /tmp
none            5.0M     0  5.0M   0% /run/lock
none            7.7G  148M  7.6G   2% /run/shm
none            100M   36K  100M   1% /run/user
```

which makes me think udev & tmpfs may be sharing RAM... hmm.

Tue Feb 27 18:10:09 PST 2018
New build started
Ran out of space again.

Wed Feb 28 10:14:46 PST 2018
Modified /etc/default/docker to set TMPDIR to /var/lib/docker/build_tmp ; created that.  Should give it plenty of space,
and I'm on SSD so it shouldn't be way too slow.

Wed Feb 28 10:53:19 PST 2018
Still building... standup slowed it down while on wifi.

Thu Mar  1 17:32:53 PST 2018
Actually finished the build yesterday. TMPDIR and docker had nothing to do with the problem. Instead, it was the Envoy build script itself using /tmp. To move it, I used:

```
sudo mkdir /var/lib/envoy
sudo chmod 777 !!$
ENVOY_DOCKER_BUILD_DIR=/var/lib/envoy ./ci/run_envoy_docker.sh './ci/do_ci.sh bazel.release'
```
