# How Arvo runs a provider

A provider is a process. Arvo starts it, reads one line from it, talks to it
over gRPC on loopback, and stops it. That is the whole lifecycle, and none of
it needs your process to know anything about Arvo.

## Starting

Arvo runs what the manifest's `run` names, from a copy of your binary in a
cache it owns — never from the checkout, so a rebuild cannot change what is
running. Two variables are set:

| Variable | What |
| --- | --- |
| `ARVO_PLUGIN_ADDR` | the address to bind, always `127.0.0.1:0` |
| `ARVO_PLUGIN_TOKEN` | a secret this spawn must require on every call |

The port is `0`, meaning the operating system chooses. There are no fixed
ports: two plugins never collide, and nothing has to be configured.

## The handshake

Bind first, then say which port you got, on **stdout**, as one line:

```
address=127.0.0.1:52341
```

Flush it. Arvo reads that line, registers the address, and starts calling.
Anything else your process writes to stdout is ignored, so logging there is
fine — but the line must arrive, or Arvo has nothing to connect to.

Bind *before* printing. A port announced before it is listening is a race
that fails only sometimes, which is worse than failing always.

## The token

The port is loopback, and anything else on the machine could still connect to
it. So every call carries the token from `ARVO_PLUGIN_TOKEN` in gRPC
metadata:

```
arvo-token: <the value of ARVO_PLUGIN_TOKEN>
```

**Refuse any call that does not present it**, with `UNAUTHENTICATED`. Compare
in constant time. Never log it.

Each spawn gets a fresh token, so a restart invalidates whatever learned the
last one. A process started by hand, with no token in its environment, may
serve without requiring one — that is how a plugin is developed — but one
Arvo started always has one.

## Credentials

Credentials never cross whole. A call that needs one carries a `Grant`: what
*that call* may use, read from Arvo's keychain and sent with the request. Use
it for that call and hold nothing.

A source needing no credential ignores the grant. One behind a sign-in reads
`bearer`, which is never a refresh token. One needing a key pair reads
`key_id` and `secret`.

A call that arrives without the grant its source needs is a call with no
session, whatever your own machine happens to hold. Do not fall back to
credentials in your environment; answer as unauthenticated and let the person
connect the account in Arvo.

## Errors

Errors cross as gRPC status codes, one per error kind, and Arvo rebuilds its
own from the code plus what it already knows it asked for. Return a status,
not a successful response describing a failure.

## Restarting and stopping

A provider that exits is restarted a few times, with a doubling pause, and
then reported unreachable with the count rather than restarted forever. A
binary that will not stay up is a fact to show someone, not a thing to spin
on.

Arvo stops what it started: when the extension is disabled, when it is
removed, and when Arvo quits. Handle termination by exiting; there is no
shutdown call, and nothing is asked of you on the way out.

## Serving more than one source

One process may serve several sources — Yahoo serves its split-adjusted and
total-return venues as two. `Describe` returns all of them; every other call
names the one it means by id.

## Developing one

Run it yourself with `ARVO_PLUGIN_ADDR=127.0.0.1:0` set and no token, watch
for your own handshake line, and point a gRPC client at the port. Arvo is not
required to develop against this; the protocol is the whole interface.
