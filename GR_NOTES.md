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
Modified /etc/default/docker to set TMPDIR to `/var/lib/docker/build_tmp`;
created that. Should give it plenty of space, and I'm on SSD so it shouldn't be
way too slow.

Wed Feb 28 10:53:19 PST 2018
Still building... standup slowed it down while on wifi.

Thu Mar  1 17:32:53 PST 2018
Actually finished the build yesterday. TMPDIR and docker had nothing to do with the problem. Instead, it was the Envoy build script itself using /tmp. To move it, I used:

```
sudo mkdir /var/lib/envoy
sudo chmod 777 !!$
ENVOY_DOCKER_BUILD_DIR=/var/lib/envoy ./ci/run_envoy_docker.sh './ci/do_ci.sh bazel.release'
```

Thu Mar  1 18:50:20 PST 2018
Forked Envoy to a ConsultingMD Github repo.
Modified my upstream to point to that.
Note that I followed the instructions in CONTRIBUTING.md and ran the
`./support/bootstrap` script, which set up a `pre-push` hook. That hook doesn't
run on my Linux install out of the box:

```
Running pre-push check; to skip this step use 'push --no-verify'
.git/hooks/pre-push: line 53: realpath: command not found
  Checking format for GR_NOTES.md
.git/hooks/pre-push: line 62: ./../../tools/check_format.py: No such file or directory
```

Ignoring that for now. I presume I'll find a link in CONTRIBUTING.md to install
the `check_format` script, and realpath is some MacOS thing that I'll have to
clone.

I'm currently running everything from the docker image, so following
support/README.md got me a way to run `check_format`. Will hand-modify pre-push to use it.

Mon Mar  5 10:29:25 PST 2018
Having trouble figuring out how to build the sample filter from inside the docker
container, so I decided to set up for native development.

Followed instructions in https://github.com/envoyproxy/envoy/blob/master/bazel/README.md#quick-start-bazel-build-for-developers
Had to add the repository and key for llvm:

```
wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key|sudo apt-key add -
```
and added `/etc/apt/sources.list.d/llvm.list` with

```
deb http://apt.llvm.org/trusty/ llvm-toolchain-trusty-5.0 main
deb-src http://apt.llvm.org/trusty/ llvm-toolchain-trusty-5.0 main
```

After `sudo apt-get update` I was finally able to `sudo apt-get install clang-format-5.0`

Also had to install `golang`

And then upgrade my g++ to support c++14. That ended up being:

```
  sudo add-apt-repository ppa:ubuntu-toolchain-r/test
  sudo apt-get update
  sudo apt-get install g\+\+-5
  sudo apt-get install gcc-4.8 g++-4.8
  sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-5 60 --slave /usr/bin/g++ g++ /usr/bin/g++-5
  sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8 60 --slave /usr/bin/g++ g++ /usr/bin/g++-4.8
```

so now I can switch back & forth using:

```
  sudo update-alternatives --config gcc
```

After *that*, I could go back to the build instructions and start with the
`bazel fetch //source/...` line.

Mon Mar  5 18:01:07 PST 2018

Once I was able to get that working I went on to build the envoy-filter-example
directory. Note that bazel doesn't cache much for you between the two
directories; it'll take the full 10m to compile & link the example filter the
first time. Afterwards, it's a bit faster, but it still takes about 30s to test
a one line change in the example http filter for me.
