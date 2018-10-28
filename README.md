SH Executor
===========

Family of functions and ports involving interacting with the system shell,
paths and external programs.

Reason
------

```erlang
> Email = "hacker+/somepath&reboot#@example.com". % this is a valid email!
> os:cmd(["mkdir -p ", Email]).
% path clobbering and a reboot may happen here!
```

Examples
--------

### Onliners

```erlang
> sh:oneliner(["touch", filename:join("/tmp/", Path)]).
{done,0,<<>>}

> sh:oneliner("uname -v"). % oneliner/1,2 funs do not include newlines
{done,0,
      <<"Darwin Kernel Version 12.4.0: Wed May  1 17:57:12 PDT 2013; root:xnu-2050.24.15~1/RELEASE_X86_64">>}

> sh:oneliner("git describe --always").
{done,128,<<"fatal: Not a valid object name HEAD">>}

> sh:oneliner("git describe --always", "/tank/proger/vxz/otp").
{done,0,<<"OTP_R16B01">>}
```

### Escaping

```erlang
> Path = sh_path:escape("email+=/subdir@example.com").
"email+=%2Fsubdir@example.com"
```

### Run

```erlang
> sh:run(["git", "clone", "https://github.com/proger/darwinkit.git"], binary, "/tmp").
{done,0,<<"Cloning into 'darwinkit'...\n">>}

> UserUrl = "https://github.com/proger/darwinkit.git".
"https://github.com/proger/darwinkit.git"
> sh:run(["git", "clone", UserUrl], binary, "/tmp").
{done,128,
      <<"fatal: destination path 'darwinkit' already exists and is not an empty directory.\n">>}

> sh:run(["ifconfig"], "/tmp/output.log", "/tank/proger/vxz/otp").
{done,0,"/tmp/output.log"}

% cat /tmp/output.log
>>> {{2013,8,28},{8,39,14}} /sbin/ifconfig
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> mtu 16384
	options=3<RXCSUM,TXCSUM>
	inet6 fe80::1%lo0 prefixlen 64 scopeid 0x1
	inet 127.0.0.1 netmask 0xff000000
	inet6 ::1 prefixlen 128
gif0: flags=8010<POINTOPOINT,MULTICAST> mtu 1280
stf0: flags=0<> mtu 1280
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	ether 7c:d1:c3:e9:24:65
	inet6 fe80::7ed1:c3ff:fee9:2465%en0 prefixlen 64 scopeid 0x4
	inet 192.168.63.163 netmask 0xfffffc00 broadcast 192.168.63.255
	media: autoselect
	status: active
p2p0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> mtu 2304
	ether 0e:d1:c3:e9:24:65
	media: autoselect
	status: inactive
>>> {{2013,8,28},{8,39,14}} exit status: 0
```

fdlink Port
-----------

Consider a case of spawning a port that does not actually
read its standard input (e.g. `socat` that bridges `AF_UNIX` with `AF_INET`):

```shell
# pstree -A -a $(pgrep make)
make run
  `-sh -c...
      `-beam.smp -- -root /usr/lib/erlang -progname erl -- -home /root -- -pa ebin -config run/sys.config -eval[ok = application:
          |-socat tcp-listen:32133,reuseaddr,bind=127.0.0.1 unix-connect:/var/run/docker.sock
          `-16*[{beam.smp}]
```

If you terminate the node, `beam` will close the port but the process
will still remain alive (thus, it will leak). To mitigate this issue,
you can use `fdlink` that will track `stdin` availability for you:

``` shell
# pstree -A -a $(pgrep make)
make run
  `-sh -c...
      `-beam.smp -- -root /usr/lib/erlang -progname erl -- -home /root -- -pa ebin -config run/sys.config -eval[ok = application:
          |-fdlink /usr/bin/socat tcp-listen:32133,reuseaddr,bind=127.0.0.1 unix-connect:/var/run/docker.sock
          |   `-socat tcp-listen:32133,reuseaddr,bind=127.0.0.1 unix-connect:/var/run/docker.sock
          `-16*[{beam.smp}]
```

### Using

```erlang
> Fdlink = sh:fdlink_executable().               % make sure your app dir is setup correctly
> Fdlink = filename:join("./priv", "fdlink").    % in case you're running directly from erlsh root
> erlang:open_port({spawn_executable, Fdlink}, [stream, exit_status, {args, ["/usr/bin/socat"|RestOfArgs]}).
```

`fdlink` will also close the standard input of its child process.

Credits
-------

* Vladimir Kirillov

OM A HUM
