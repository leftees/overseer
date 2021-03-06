# overseer

[![GoDoc](https://godoc.org/github.com/jpillora/overseer?status.svg)](https://godoc.org/github.com/jpillora/overseer)

`overseer` is a library for creating monitorable, gracefully restarting, self-upgrading binaries in Go (golang). The main goal of this project is to facilitate the creation of self-upgrading binaries which play nice with standard process managers, secondly it should expose a simple API with reasonable defaults.

![overseer diagram](https://docs.google.com/drawings/d/1o12njYyRILy3UDs2E6JzyJEl0psU4ePYiMQ20jiuVOY/pub?w=566&h=284)

Commonly, graceful restarts are performed by the active process (*dark blue*) closing its listeners and passing these matching listening socket files (*green*) over to a newly started process. This restart causes any **foreground** process monitoring to incorrectly detect a program crash. `overseer` attempts to solve this by using a small process to perform this socket file exchange and proxying signals and exit code from the active process.

:warning: *This is beta software. Do not use this in production yet. Consider the API unstable. Please report any [issues](https://github.com/jpillora/overseer) you encounter.*

### Features

* Simple
* Works with process managers
* Graceful, zero-down time restarts
* Easy self-upgrading binaries

### Install

```sh
go get github.com/jpillora/overseer
```

### Quick example

This program works with process managers, supports graceful, zero-down time restarts and self-upgrades its own binary.

``` go
package main

import (
	"fmt"
	"log"
	"net/http"
	"time"

	"github.com/jpillora/overseer"
	"github.com/jpillora/overseer/fetcher"
)

//create another main() to run the overseer process
//and then convert your old main() into a 'prog(state)'
func main() {
	overseer.Run(overseer.Config{
		Program: prog,
		Address: ":3000",
		Fetcher: &fetcher.HTTP{
			URL:      "http://localhost:4000/binaries/myapp",
			Interval: 1 * time.Second,
		},
	})
}

//prog(state) runs in a child process
func prog(state overseer.State) {
	log.Printf("app (%s) listening...", state.ID)
	http.Handle("/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "app (%s) says hello\n", state.ID)
	}))
	http.Serve(state.Listener, nil)
}
```

**How it works:**

* `overseer` uses the main process to check for and install upgrades and a child process to run `Program`.
* The main process retrieves the files of the listeners described by `Address/es`.
* The child process is provided with these files which is converted into a `Listener/s` for the `Program` to consume.
* All child process pipes are connected back to the main process.
* All signals received on the main process are forwarded through to the child process.
* `Fetcher` runs in a goroutine and checks for updates at preconfigured interval. When `Fetcher` returns a valid binary stream (`io.Reader`), the master process saves it to a temporary location, verifies it, replaces the current binary and initiates a graceful restart.
* The `fetcher.HTTP` accepts a `URL`, it polls this URL with HEAD requests and until it detects a change. On change, we `GET` the `URL` and stream it back out to `overseer`. See also `fetcher.S3`.
* Once a binary is received, it is run with a simple echo token to confirm it is a `overseer` binary.
* Except for scheduled restarts, the active child process exiting will cause the main process to exit with the same code. So, **`overseer` is not a process manager**.

See [Config](https://godoc.org/github.com/jpillora/overseer#Config)uration options [here](https://godoc.org/github.com/jpillora/overseer#Config) and the runtime [State](https://godoc.org/github.com/jpillora/overseer#State) available to your program [here](https://godoc.org/github.com/jpillora/overseer#State).

### More examples

* See the [example/](example/) directory and run `example.sh`, you should see the following output:

	```sh
	$ cd example/
	$ sh example.sh
	serving . on port 5002
	BUILT APP (1)
	RUNNING APP
	app#1 (1cd8b9928d44b0a6e89df40574b8b6d20a417679) listening...
	app#1 (1cd8b9928d44b0a6e89df40574b8b6d20a417679) says hello
	app#1 (1cd8b9928d44b0a6e89df40574b8b6d20a417679) says hello
	BUILT APP (2)
	app#2 (b9b251f1be6d0cc423ef921f107cb4fc52f760b3) listening...
	app#2 (b9b251f1be6d0cc423ef921f107cb4fc52f760b3) says hello
	app#2 (b9b251f1be6d0cc423ef921f107cb4fc52f760b3) says hello
	app#1 (1cd8b9928d44b0a6e89df40574b8b6d20a417679) says hello
	app#1 (1cd8b9928d44b0a6e89df40574b8b6d20a417679) exiting...
	BUILT APP (3)
	app#3 (248f80ea049c835e7e3714b7169c539d3a4d6131) listening...
	app#3 (248f80ea049c835e7e3714b7169c539d3a4d6131) says hello
	app#3 (248f80ea049c835e7e3714b7169c539d3a4d6131) says hello
	app#2 (b9b251f1be6d0cc423ef921f107cb4fc52f760b3) says hello
	app#2 (b9b251f1be6d0cc423ef921f107cb4fc52f760b3) exiting...
	app#3 (248f80ea049c835e7e3714b7169c539d3a4d6131) says hello
	```

	**Note:** `app#1` stays running until the last request is closed.

* Only use graceful restarts:

	```go
	func main() {
		overseer.Run(overseer.Config{
			Program: prog,
			Address: ":3000",
		})
	}
	```

	Send `main` a `SIGUSR2` (`Config.RestartSignal`) to manually trigger a restart

* Only use auto-upgrades, no restarts

	```go
	func main() {
		overseer.Run(overseer.Config{
			Program: prog,
			NoRestartAfterFetch: true
			Fetcher: &fetcher.HTTP{
				URL:      "http://localhost:4000/binaries/myapp",
				Interval: 1 * time.Second,
			},
		})
	}
	```

	Your binary will be upgraded though it will require manual restart from the user, suitable for creating self-upgrading command-line applications.

### Known issues

* The master process's `overseer.Config` cannot be changed via an upgrade, the master process must be restarted.
	* Therefore, `Addresses` can only be changed by restarting the main process.
* Currently shells out to `mv` for moving files because `mv` handles cross-partition moves unlike `os.Rename`.
* Only supported on darwin and linux.

### More documentation

* [Core `overseer` package](https://godoc.org/github.com/jpillora/overseer)
* [Common `fetcher.Interface`](https://godoc.org/github.com/jpillora/overseer/fetcher#Interface)
* [HTTP fetcher type](https://godoc.org/github.com/jpillora/overseer/fetcher#HTTP)
* [S3 fetcher type](https://godoc.org/github.com/jpillora/overseer/fetcher#S3)

### Docker

1. Compile your `overseer`able `app` to a `/path/on/docker/host/dir/app`
1. Then run it with:

	```sh
	docker run -d -v /path/on/docker/host/dir/:/home/ -w /home/ debian  -w /home/app
	```

1. For testing, swap out `-d` (daemonize) for `--rm -it` (remove on exit, input, terminal)
1. `app` can use the current working directory as storage
1. If the OS doesn't ship with TLS certs, you can mount them from the host with `-v /etc/ssl/certs/ca-certificates.crt:/etc/ssl/certs/ca-certificates.crt`

### TODO

* Tests! The test suite should drive an:
 	* HTTP client for verifying application version
	* HTTP server for providing application upgrades
	* an `overseer` process via `exec.Cmd`
* Slave process should pass new config back to master to:
	* Update logging settings
	* Update socket bindings
* Fetchers
	* HTTP fetcher long-polling (pseduo-push)
	* SCP fetcher (connect to a server, poll path)
	* Github fetcher (given a repo, poll releases)
	* etcd fetcher (given a cluster, watch key)
* `overseer` CLI tool ([TODO](cmd/overseer/TODO.md))
* `overseer` package
	* Execute and verify calculated delta updates with https://github.com/kr/binarydist
	* [Omaha](https://coreos.com/docs/coreupdate/custom-apps/coreupdate-protocol/) client support
