# Metronome

## Description

Metronome is an interval scheduler and task runner.  It can be used
locally as a cron replacement, or as a network-wide job executor.

Events are stored via simple database rows, and optionally managed
via AMQP events.  Interval/time values are expressed with intuitive
English phrases, ie.: 'at 2pm', or 'Starting in 20 minutes, run every 10
seconds and then finish in 2 days', or 'execute 12 times during the next
minute'.

## Synopsis

Here's an example of a simple cron clone:

```
#!ruby

require 'symphony/metronome'

Symphony.load_config

Symphony::Metronome.run do |opts, id|
	Thread.new do
		pid = fork { exec opts.delete('command') }
		Process.waitpid( pid )
	end
end
```


And here's a simplistic AMQP message broadcaster, using existing
Symphony connection information:

```
#!ruby

require 'symphony/metronome'

Symphony.load_config

Symphony::Metronome.run do |opts, id|
	key = opts.delete( 'routing_key' ) or next
	exchange = Symphony::Queue.amqp_exchange
	exchange.publish( 'hi from Metronome!', routing_key: key )
end
```

## Adding Actions

There are two primary components to Metronome -- getting actions into
its database, and performing some task with those actions when the time
is appropriate.

By default, Metronome will start up an AMQP listener, attached to your
Symphony exchange, and wait for new scheduling messages.  There are two
events it will take action on:

metronome.create:

	Create a new scheduled event.  The payload should be a hash.  An
	'expression' key is required, that provides the interval description.
	Anything additional is serialized to 'options', that are passed to the
	block when the interval fires.  You can populate it with anything
	your task requires to execute.

metronome.delete:

	The payload is the row ID of the action.  Metronome removes it from
	the database.

If you'd prefer not to use the AMQP listener, you can put actions into
Metronome using any database methodology you please.  When the daemon
starts up or receives a HUP signal, it will re-read and schedule out
upcoming work.


## Options

Metronome uses
[Configurability](https://rubygems.org/gems/configurability) to determine
behavior.  The configuration is a [YAML](http://www.yaml.org/) file.  It
shares AMQP configuration with Symphony, and adds metronome specific
controls in the 'metronome' key.

	metronome:
		splay: 0
		listen: true
		db: sqlite:///tmp/metronome.db


### splay

Randomize all start times for actions by this many seconds on either
side of the original execution time.  Defaults to none.

### listen

Start up an AMQP listener using Symphony configuration, for remote
administration of schedule events.  Defaults to true.

### db

A [Sequel](https://rubygems.org/gems/sequel) connection URI.  Currently,
Metronome is tested under SQLite and PostgreSQL.  Defaults to a SQLite
file at /tmp/metronome.db.


## Scheduling Examples

Note that Metronome is designed as an interval scheduler, not a
calendaring app.  It doesn't have any concepts around phrases like "next
tuesday", or "the 3rd sunday after christmas".  If that's what you're
after, check out the [chronic](http://rubygems.org/gems/chronic)
library instead.

Here are a small set of example expressions.  Feel free to use the
*metronome-exp* utility to get a feel for what Metronome anticipates.

```
in 30.5 minutes
once an hour
every 15 minutes for 2 days
at 2014-05-01
at 2014-04-01 14:00:25
at 2pm
starting at 2pm once a day
start in 1 hour from now run every 5 seconds end at 11:15pm
every other hour
run every 7th minute for a day
once a day ending in 1 week
run once a minute for an hour starting in 6 days
10 times a minute for 2 days
run 45 times every hour
30 times per day
start at 2010-01-02 run 12 times and end on 2010-01-03
starting in an hour from now run 6 times a minute for 2 hours
beginning a day from now, run 30 times per minute and finish in 2 weeks
execute 12 times during the next 2 minutes
once a minute beginning in 5 minutes
```

In general, you can use reasonably intuitive phrasings.  Capitalization,
whitespace, and punctuation doesn't matter.  When describing numbers,
use digit/integer form instead of words, ie: '1 hour' instead of 'one
hour'.


## Installation

```
gem install symphony-metronome
```

## Contributing

You can check out the current development source with Mercurial
[here](http://code.martini.nu/symphony-metronome), or via a mirror:

 * github: https://github.com/mahlonsmith/Symphony-Metronome
 * SourceHut: https://hg.sr.ht/~mahlon/Symphony-Metronome

After checking out the source, run:

```
$ rake
```

This task will run the tests/specs and generate API documentation.

If you use [rvm](http://rvm.io/), entering the project directory will
install any required development dependencies.
