# uphold/bitcoin-abc

A Bitcoin ABC docker image.

[![uphold/bitcoin-abc][docker-pulls-image]][docker-hub-url] [![uphold/bitcoin-abc][docker-stars-image]][docker-hub-url] [![uphold/bitcoin-abc][docker-size-image]][docker-hub-url] [![uphold/bitcoin-abc][docker-layers-image]][docker-hub-url]

## Tags

- `0.20.2-alpine`, `0.20-alpine`, `alpine` ([0.20/alpine/Dockerfile](https://github.com/uphold/docker-bitcoin-abc/blob/master/0.20/alpine/Dockerfile))
- `0.19.5-alpine`, `0.19-alpine` ([0.19/alpine/Dockerfile](https://github.com/uphold/docker-bitcoin-abc/blob/master/0.19/alpine/Dockerfile))
- `0.18.8-alpine`, `0.18-alpine` ([0.18/alpine/Dockerfile](https://github.com/uphold/docker-bitcoin-abc/blob/master/0.18/alpine/Dockerfile))
- `0.17.2-alpine`, `0.17-alpine` ([0.17/alpine/Dockerfile](https://github.com/uphold/docker-bitcoin-abc/blob/master/0.17/alpine/Dockerfile))
- `0.16.2-alpine`, `0.16-alpine` ([0.16/alpine/Dockerfile](https://github.com/uphold/docker-bitcoin-abc/blob/master/0.16/alpine/Dockerfile))
- `0.15.1-alpine`, `0.15-alpine` ([0.15/alpine/Dockerfile](https://github.com/uphold/docker-bitcoin-abc/blob/master/0.15/alpine/Dockerfile))

- `0.20.2`, `0.20`, `latest` ([0.20/Dockerfile](https://github.com/uphold/docker-bitcoin-abc/blob/master/0.18/Dockerfile))
- `0.19.5`, `0.19` ([0.19/Dockerfile](https://github.com/uphold/docker-bitcoin-abc/blob/master/0.19/Dockerfile))
- `0.18.8`, `0.18` ([0.18/Dockerfile](https://github.com/uphold/docker-bitcoin-abc/blob/master/0.18/Dockerfile))
- `0.17.2`, `0.17` ([0.17/Dockerfile](https://github.com/uphold/docker-bitcoin-abc/blob/master/0.17/Dockerfile))
- `0.16.2`, `0.16` ([0.16/Dockerfile](https://github.com/uphold/docker-bitcoin-abc/blob/master/0.16/Dockerfile))
- `0.15.1`, `0.15` ([0.15/Dockerfile](https://github.com/uphold/docker-bitcoin-abc/blob/master/0.15/Dockerfile))

## What is Bitcoin ABC?

Bitcoin ABC is a full node implementation of the [Bitcoin Cash](https://www.bitcoincash.org) protocol. With a future roadmap of massive scaling, Bitcoin ABC allows an immediate block size increase with a simple, sensible, adjustable blocksize cap. Learn more about [Bitcoin ABC](https://www.bitcoinabc.org).

## Usage

### How to use this image

This image contains the main binaries from the Bitcoin ABC project - `bitcoind`, `bitcoin-cli` and `bitcoin-tx`. It behaves like a binary, so you can pass any arguments to the image and they will be forwarded to the `bitcoind` binary:

```sh
❯ docker run --rm -it uphold/bitcoin-abc \
  -printtoconsole \
  -regtest=1 \
  -rpcallowip=172.17.0.0/16 \
  -rpcpassword=bar \
  -rpcuser=foo
```

By default, `bitcoind` will run as user `bitcoin` for security reasons and with its default data dir (`~/.bitcoin/`). If you'd like to customize where `bitcoin-abc` stores its data, you must use the `BITCOIN_ABC_DATA` environment variable. The directory will be automatically created with the correct permissions for the `bitcoin` user and `bitcoin-abc` automatically configured to use it.

```sh
❯ docker run --env BITCOIN_ABC_DATA=/var/lib/data --rm -it uphold/bitcoin-abc \
  -printtoconsole \
  -regtest=1
```

You can also mount a directory it in a volume under `/home/bitcoin/.bitcoin` in case you want to access it on the host:

```sh
❯ docker run -v ${PWD}/data:/home/bitcoin/.bitcoin -it --rm uphold/bitcoin-abc \
  -printtoconsole \
  -regtest=1
```

You can optionally create a service using `docker-compose`:

```yml
bitcoin-abc:
  image: uphold/bitcoin-abc
  command:
    -printtoconsole
    -regtest=1
```

### Using RPC to interact with the daemon

There are two communications methods to interact with a running Bitcoin ABC daemon.

The first one is using a cookie-based local authentication. It doesn't require any special authentication information as running a process locally under the same user that was used to launch the Bitcoin ABC daemon allows it to read the cookie file previously generated by the daemon for clients. The downside of this method is that it requires local machine access.

The second option is making a remote procedure call using a username and password combination. This has the advantage of not requiring local machine access, but in order to keep your credentials safe you should use the newer `rpcauth` authentication mechanism.

#### Using cookie-based local authentication

Start by launching the Bitcoin ABC daemon:

```sh
❯ docker run --rm --name bitcoin-abc-server -it uphold/bitcoin-abc \
  -printtoconsole \
  -regtest=1
```

Then, inside the running `bitcoin-abc-server` container, locally execute the query to the daemon using `bitcoin-cli`:

```sh
❯ docker exec --user bitcoin bitcoin-abc-server bitcoin-cli -regtest getmininginfo

{
  "blocks": 0,
  "currentblocksize": 0,
  "currentblockweight": 0,
  "currentblocktx": 0,
  "difficulty": 4.656542373906925e-10,
  "errors": "",
  "networkhashps": 0,
  "pooledtx": 0,
  "chain": "regtest"
}
```

In the background, `bitcoin-cli` read the information automatically from `/home/bitcoin/.bitcoin/regtest/.cookie`. In production, the path would not contain the regtest part.

#### Using rpcauth for remote authentication

Before setting up remote authentication, you will need to generate the `rpcauth` line that will hold the credentials for the Bitcoind Core daemon. You can either do this yourself by constructing the line with the format `<user>:<salt>$<hash>` or use the official `rpcuser.py` script to generate this line for you, including a random password that is printed to the console.

Example:

```sh
❯ curl -sSL https://raw.githubusercontent.com/bitcoin/bitcoin/master/share/rpcuser/rpcuser.py | python - <username>

String to be appended to bitcoin.conf:
rpcauth=foo:7d9ba5ae63c3d4dc30583ff4fe65a67e$9e3634e81c11659e3de036d0bf88f89cd169c1039e6e09607562d54765c649cc
Your password:
qDDZdeQ5vw9XXFeVnXT4PZ--tGN2xNjjR4nrtyszZx0=
```

Note that for each run, even if the username remains the same, the output will be always different as a new salt and password are generated.

Now that you have your credentials, you need to start the Bitcoin ABC daemon with the `-rpcauth` option. Alternatively, you could append the line to a `bitcoin.conf` file and mount it on the container.

Let's opt for the Docker way:

```sh
❯ docker run --rm --name bitcoin-abc-server -it uphold/bitcoin-abc \
  -printtoconsole \
  -regtest=1 \
  -rpcallowip=172.17.0.0/16 \
  -rpcauth='foo:e1fcea9fb59df8b0388f251984fe85$26431097d48c5b6047df8dee64f387f63835c01a2a463728ad75087d0133b8e6'
```

Two important notes:

1. Some shells require escaping the rpcauth line (e.g. zsh), as shown above.
2. It is now perfectly fine to pass the rpcauth line as a command line argument. Unlike `-rpcpassword`, the content is hashed so even if the arguments would be exposed, they would not allow the attacker to get the actual password.

You can now connect via `bitcoin-cli` or any other [compatible client](https://github.com/uphold/bitcoin-abc). You will still have to define a username and password when connecting to the Bitcoin ABC RPC server.

To avoid any confusion about whether or not a remote call is being made, let's spin up another container to execute `bitcoin-cli` and connect it via the Docker network using the password generated above:

```sh
❯ docker run --link bitcoin-abc-server --rm uphold/bitcoin-abc bitcoin-cli -rpcconnect=bitcoin-abc-server -regtest -rpcuser=foo -rpcpassword='j1DuzF7QRUp-iSXjgewO9T_WT1Qgrtz_XWOHCMn_O-Y=' getmininginfo

{
  "blocks": 0,
  "currentblocksize": 0,
  "currentblockweight": 0,
  "currentblocktx": 0,
  "difficulty": 4.656542373906925e-10,
  "errors": "",
  "networkhashps": 0,
  "pooledtx": 0,
  "chain": "regtest"
}
```

Done!

## Images

The `uphold/bitcoin-abc` image comes in multiple flavors:

### `uphold/bitcoin-abc:latest`

Points to the latest release available of Bitcoin ABC. Occasionally pre-release versions will be included.

### `uphold/bitcoin-abc:<version>`

Based on a slim Debian image, targets a specific version branch or release of Bitcoin ABC.

### `uphold/bitcoin-abc:<version>-alpine`

Based on Alpine Linux with Berkeley DB 4.8 (cross-compatible build), targets a specific version branch or release of Bitcoin ABC.

## Supported Docker versions

This image is officially supported on Docker version 17.09.0-ce, with support for older versions provided on a best-effort basis.

## License

[License information](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/master/COPYING) for the software contained in this image.

[License information](https://github.com/uphold/docker-bitcoin-abc/blob/master/LICENSE) for the [uphold/docker-bitcoin-abc][docker-hub-url] docker project.

[docker-hub-url]: https://hub.docker.com/r/uphold/bitcoin-abc
[docker-layers-image]: https://img.shields.io/imagelayers/layers/uphold/bitcoin-abc/latest.svg?style=flat-square
[docker-pulls-image]: https://img.shields.io/docker/pulls/uphold/bitcoin-abc.svg?style=flat-square
[docker-size-image]: https://img.shields.io/imagelayers/image-size/uphold/bitcoin-abc/latest.svg?style=flat-square
[docker-stars-image]: https://img.shields.io/docker/stars/uphold/bitcoin-abc.svg?style=flat-square
