+++
title = "A parallel quest in elixir"
date = 2020-03-31
template = "blog_post.html"
[taxonomies]
tags = ["elixir"]
+++

Since August 2018, EBANX has been running a project to map and
document its accounting and financial flows. The main goal is to
provide visibility to processes such as payments, refunds,
chargebacks, etc.

We've been working on this system, which basically is an ETL in
Elixir that takes information from our transactional database and
converts it into financial and accounting events, persisting them
on an accounting database (more like the accounting book format.)

For example, for a payment requested and confirmed on our
transactional table payment, two accounting entries (credit and
debit) will be required, and we identify those with a
`PAYMENT_CONFIRMED|PAYMENT_RECOGNITION` event on our accounting
database.

The amount of data to be processed with this pipeline can be
HUGE. We sometimes have import tasks involving dozens of millions
of ids that request data from over twenty database tables to
accommodate all variations of product peculiarities. It's not
rare to have jobs lasting something about 6 to 8 hours in a 36
cores machine.

Then at some point we decided the pipeline internals was to
complicated and inaccessible for most part of the team that were
not really Elixir experts (darn macros).

I then came forth with a new version and started implementing it
and replacing the old pipeline. Also had to wanted to substitute
the code for the importing job to work better with the new code.

In the exact day we have all components placed and tests passing
with the new code comes the news: the project scope has changed,
it needed to analyze a new period of time and we needed all the
new data imported before we could even start working.

Not really how I had planned to test the new code, but it had to
be done and there was not really an easy way to put the old
pipeline working again (my bad).

It's ok, no pressure.

So I put the new ids file to import and the time metrics says the
processing would be finished in days, not hours. It would be a
big blunder if I had done all that work and in the end have to
revert everything for not having given though on the performance.

Oh, of course, I just forgot something...

"Let's make it parallel", I said, "we've done it before". Yeah,
if just the elixir runtime would agree with me.

The problem was not so simple, we had a previous version of the
same importing pipeline which we used to run on Flow but we would
like to try something simpler like
[`Task.async_stream`](https://hexdocs.pm/elixir/1.9.1/Task.html#async_stream/3).

We had the new sub-pipeline that would do all of the important
work. In the importing pipeline (that will be my main subject
here) we just had to feed it with chunks of ids and stream the
results to the Database component.

The internal pipeline was already complex enough so it was not a
very good candidate for parallelism and we didn't want to
complicate our logic there.

So an easy way out is to parallelize the calls to the internal
pipeline on the importing job pipeline that will also persist the
information on the database.

Here is some draft code for you to have a taste:

```Elixir
defmodule MyProj.FileImporterJob do
  alias MyProj.Database
  alias MyProj.EventGeneratorPipeline
  @load_chunk_size 50
  @insert_chunk_size 3
  @load_workers 8

  @insert_workers 4
  def execute_pipeline(ids_file_path) do
    ids_file_path
    |> File.stream!(read_ahead: 65536)
    |> Stream.map(&String.trim/1)
    |> Stream.map(&String.to_integer/1)
    |> Stream.chunk_every(@load_chunk_size)
    |> Task.async_stream(&EventGeneratorPipeline.execute/1, max_concurrency: @load_workers)
    |> Stream.flat_map(&ensure_and_forward_stream/1)
    |> Stream.chunk_every(@insert_chunk_size)
    |> Task.async_stream(&Database.insert/1, max_concurrency: @insert_workers)
    |> Stream.map(&ensure_and_forward_stream/1)
    |> Stream.run
  end
  defp ensure_and_forward_stream({:ok, stream}), do: stream
end
```

Version 1 of the new `FileImporterJob` using the new module `EventGeneratorPipeline`

The code was easy to read and all constraints I wanted to put on
it were perfectly clear: how many records would I get from the
database per query, how many concurrent database connections, etc.

So, I get to run this code and the time expectation is absurdly
bigger than it should be. A `htop` reveals to me that only 1 of our
36 available cores was effectively doing some work. To complete
the situation, even that one core I had working would from time
to time drop to almost zero.

What has gone wrong?

We tried a lot of hypotheses, I remember looking for the queue
size of an internal genserver
[to see if that was the bottleneck](https://www.cogini.com/blog/avoiding-genserver-bottlenecks/).
But it wasn't.

Desperation mode: `on`.

I put a `:read_ahead` option on the `File.stream` trying to make a
better use of my disk and there was some improvement. Mainly the
problem regarding drops in the processing became less frequent
and less severe, but this surely was not my only bottleneck.

At some point, trying a lot of different things, I removed the
code between the two parallel parts, and BOOM: things started
working the way they were supposed to. Or at least, I got a
steady 65%~80% usage on all cores (which I trust is good enough
for a system that depends heavily on the database on the two ends.
) In the end, the parallelism problem was a little sequential
line of code throttling the whole pipeline. And that was not
obvious, unfortunately.


```Elixir
defmodule MyProj.FileImporterJob do
  alias MyProj.Database
  alias MyProj.EventGeneratorPipeline
  @load_chunk_size 50
  @insert_chunk_size 3
  @load_workers 8

  @insert_workers 4
  def execute_pipeline(ids_file_path) do
    ids_file_path
    |> File.stream!(read_ahead: 65536)
    |> Stream.map(&String.trim/1)
    |> Stream.map(&String.to_integer/1)
    |> Stream.chunk_every(@load_chunk_size)
    |> Task.async_stream(&EventGeneratorPipeline.execute/1, max_concurrency: @load_workers)
    |> Task.async_stream(&Database.insert(&1, insert_chunk_size: @insert_chunk_size), max_concurrency: @insert_workers)
    |> Stream.map(&ensure_and_forward_stream/1)
    |> Stream.run
  end
  defp ensure_and_forward_stream({:ok, stream}), do: stream
end
```


This also made me realize something important. The code was not
executing the way I thought it would. In fact, seeing from a low
level perspective, all a method of the Stream module does is to
define a function, and this is not executed until someone (the
next piped function) calls that hidden function asking for a new
value. So the flow is not given by the first line pushing data
downwards, but by the last line asking up for the next value and
cascading up this call. That means that unlike what I thought
initially, it was not the first parallel part that was not being
able to pass down the values in the speed it wanted. The truth
was that it was not receiving work to do since the throttle below
was not calling it frequently enough.

That was a hard realization to have, the data flow is surely
top-down, but in some sense the execution flow is bottom-up and
then top-down. Requests go up before the data can come down, and
the requests might get throttled somewhere before the data even
has a chance to walk down the pipeline.

And overall it makes sense, that flatten was a sequential
operation after all and removing it should improve the data flow
from one step to the other since a parallel step would be capable
of inducing more demand from its previous step.


```Elixir
defmodule MyProj.FileImporterJob do
  alias MyProj.Database
  alias MyProj.EventGeneratorPipeline

  @load_chunk_size 50
  @insert_chunk_size 3
  @load_workers 8
  @insert_workers 4

  def execute_pipeline(ids_file_path) do
    # we also decided to get all values to memory up-front
    # and avoid the disk would mess our performance visibility
    specs =
      ids_file_path
      |> File.stream!(read_ahead: 65536)
      |> Stream.map(&String.trim/1)
      |> Stream.map(&String.to_integer/1)
      |> Enum.chunk_every(@load_chunk_size)

    specs
    |> Task.async_stream(&EventGeneratorPipeline.execute/1, max_concurrency: @load_workers)
    |> Task.async_stream(&Database.insert(&1, insert_chunk_size: @insert_chunk_size), max_concurrency: @insert_workers)
    |> Stream.run
  end
end
```

There is one final problem with this approach: inter process
communication.

The big data chunks in this whole process are actually generated
in the module `EventGeneratorPipeline` that runs inside the
processes `Task.async_stream` spawns, but they are consumed in the
other parallel part of the code, in another context. Which means,
as far as I know, that they will be copied in memory to be passed
to these others processes. And this copy could be avoided if we
put producer and consumer to run in the same context:

```Elixir
defmodule MyProj.FileImporterJob do
  alias MyProj.Database
  alias MyProj.EventGeneratorPipeline

  @load_chunk_size 50
  @insert_chunk_size 3
  @workers 8

  def execute_pipeline(ids_file_path) do
    specs =
      ids_file_path
      |> File.stream!(read_ahead: 65536)
      |> Stream.map(&String.trim/1)
      |> Stream.map(&String.to_integer/1)
      |> Enum.chunk_every(@load_chunk_size)

    specs
    |> Task.async_stream(&process_chunk/1, max_concurrency: @workers)
    |> Stream.run
  end
  def process_chunk(chunk) do
    chunk
    |> EventGeneratorPipeline.execute
    |> Database.insert(insert_chunk_size: @insert_chunk_size)
  end
end
```

This had a massive improvement in performance. The execution time
of the whole job for a 1 million ids dropped from ~30 minutes to
~15 minutes, roughly 50% improvement. I really regret not having
done this earlier.

But it didn't come without a cost to the code. Previously I had
the impression that it could both be loading data from the source
database, and inserting at my final database in an independent
fashion. In a way I could keep data flowing in and out at the
maximum possible speed for each one.

But that is not possible anymore using the new version, although
if I just let this run for enough time I think the threads would
eventually end up coordinating themselves, while some of them are
fetching data some others would be storing.

This last version of the pipeline was a trade-off. To get less
memory copy I lost a whole bunch of control on my code. You may
see in this last example that we only have a workers variable, no
more load_workers and insert_workers. So I no longer have fine
control on how many workers I have using each database at any
given time, just a general number of workers that may be
distributing well the workload or may be hitting the same
database all at once causing usage spikes.

But my overall performance went up, so I think this was a small
price to pay.


## Conclusions

Elixir has been a wonderful language to work with. It was not my
first language when I started in this project, but it has surely
won a lot of my appreciation since then. But I still learn a lot
about the language every time something comes up and wanted to
share this story for the one's who will try to do something
parallel in elixir.

Some things where really great, like how parallel and non
parallel code can interact in elixir without losing that nice
chain of pipes look. Somethings ended up not so great when things
didn't actually worked so well as I had imagined.

If I were to summarize in one thing to take from this post, most
of the mistakes were about relying on elixir's nice syntax and
easy of use without really given thought to what where the hard
science concepts behind that commands.

Like the Inter-Process-Communication situation. I didn't thought
at first that getting data in and out that function would require
me a overhead so big as copying everything.

Or the "stream is just a function declaration" thing, that got
me off guard. At some point I had 40 workers calling the internal
pipeline, but that is made of streams internally, I was also not
calling anything to materialize those results in that context. So
what I most probably had was 40 workers defining functions
without really doing any work.

Also, something I took from this episode is: even in a language
so nice as elixir I still had to sacrifice usability and
readability for performance.

That's all folks.

If anything is amiss or not accurate let me know. Although this
situation is good enough for all we use, it would be nice to
revisit this subject and improve upon it.
