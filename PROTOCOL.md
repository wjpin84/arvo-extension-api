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

## What you serve

Two services, at the address you announced:

| Service | Why |
| --- | --- |
| `arvo.plugin.v1.Plugin` | `GetManifest`, how Arvo learns what you are and what you can do |
| whatever you named in the manifest | the work itself |

`GetManifest` answers with an id, a name, a version, and a **capability per
service you serve**, named exactly as the service is named:
`arvo.source.v1.Source`, `arvo.signal.v1.Signals`. Arvo asks nothing of a
service you did not claim, so a capability you omit is a capability you do
not have.

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

## Publishing signals

A provider naming `arvo.signal.v1.Signals` publishes named values that may be
absent — a regime classifier is the first one, publishing labels rather than
numbers.

`Describe` says what you publish: a name in the open dotted namespace, a line
about what it means, whether it is **causal**, and what computes it.

Causal means each value was computed from data available at that instant. It
is your claim to make, and it decides what a study may do with the series:
gate a backtest on a causal signal, describe a result with a descriptive one.
A label computed over a whole window after the fact is look-ahead of the most
flattering kind, so if that is what yours is, say `causal: false` and say why
in `because`. That sentence is what Arvo shows the person whose backtest it
refuses.

`Latest` says what you think right now, and this is where the invariant
lives:

- **A value you do not have is absent.** Send the point with no value. Never
  a zero, which is a number a threshold matches.
- **Never repeat the last value you knew** once it is no longer current. A
  rule reading a stale "the market is trending" goes on believing it.
- A provider that is down says nothing at all, and Arvo drops its names from
  the namespace rather than remembering them. A predicate over an absent
  value is false, so a rule over a dead classifier's signal does not fire.

Arvo polls rather than subscribing. A poll that fails is absence, which this
protocol already has a word for — a broken stream would be a third state
meaning the same thing.

## Serving more than one source

One process may serve several sources — Yahoo serves its split-adjusted and
total-return venues as two. `Describe` returns all of them; every other call
names the one it means by id.

## Developing one

Run it yourself with `ARVO_PLUGIN_ADDR=127.0.0.1:0` set and no token, watch
for your own handshake line, and point a gRPC client at the port. Arvo is not
required to develop against this; the protocol is the whole interface.
