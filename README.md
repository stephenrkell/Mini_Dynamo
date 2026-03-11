This lab session is about experimenting with a "realistic" but still
reasonably simple quorum-based replicated storage system -- a Python
simulation of Amazon's influential Dynamo system that was first
published in 2007. The following paper describes how Dynamo uses
quorums.

https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf

Read (at least) sections 1, 4.3, 4.5 (discussing quorums) and 6
(initial part, before 6.1) of the paper.

In particular note that it can allow reduced read quorums, where R +
W <= N, in some cases -- unlike the strong (one-copy
serializability) property we studied in lectures. What might the
benefit of this be? What about the drawbacks?

Now examine the Python simulation in this repository, which is
a stripped-down version of the Dynamo system, created by
Raahul Krishna Durairaj (upstream: https://github.com/RAAHUL-tech/Mini_Dynamo).

If using a lab machine you will have to create a Python virtual
environment to download the dependencies of the simulation.

```
mkdir venv
python3 -m venv `pwd`/venv
./venv/bin/pip3 install flask
./venv/bin/pip3 install requests
```

Then you can follow the recipes in the README.upstream.md, except that where
it uses `python`, use `./venv/bin/python3`. Or from your
own machine you should be able to use `python` or `python3`
directly, as long as you have the `flask` and `requests` packages
installed.

Check you can run the examples in the README.upstream.md that start up the
"3-node cluster". You can use one terminal per node, or you can do
everything in one terminal if you start nodes in the background
using '&'. (If you do this, `jobs -l` is a useful command.)

Work through the failure simulations and check you can reproduce
them. E.g. you should see that if you have killed two nodes, you can
no longer successfully write. What about reading?

Considering the "Design trade-offs" table, two negatives in the
right-hand column stand out: "stale reads possible" and "client must
handle conflicts". Can you write some Python or shell scripts that
demo these issues? Specifically, can you get the system to
perform a stale read, and can you get it to perform two conflicting
writes such that both answers will be reported to a reading client?

(The latter response should look like the one seen under the
"Response (with conflicts)" heading.)

It may be useful to know that you can pause, rather than just kill, a client.
For example,

```
kill -STOP 23456
```

will pause the process with PID 23456
You can resume it later with

```
kill -CONT 23456
```

This may help you create a conflicting pair of writes.

To get the PID of a process you can use `jobs -l` (if using just one terminal)
or `ps xf | grep 'node\.py'` (even if using many terminals).

Can you get a stale read if you set R=2? What about if R=1?

Have a look through the implementation's Python code.  Although the
system claims to be "eventually consistent", something is missing
from the implementation... can you tell what it is? This may also be
evident once you have done a conflicting write and then tried to
read the value... and then tried again later....
