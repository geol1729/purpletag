# purpletag

A tool to track polarized hashtags used by members of the U.S. congress.

## Install

`pip install purpletag`

or, from source:

```
git clone https://github.com/casmlab/purpletag.git
cd purpletag
python setup.py install
```

## Configuration

purpletag depends on [twutil](https://github.com/tapilab/twutil) for
collecting data from Twitter. You'll need to put your credentials in the
following environmental variables:

- `TW_CONSUMER_KEY`
- `TW_CONSUMER_SECRET`
- `TW_ACCESS_TOKEN`
- `TW_ACCESS_TOKEN_SECRET`

purpletag also depends on a configuration file (see [`sample.cfg`](sample.cfg)
for an example). By default, it is assumed to be in `~/.purpletag`, but you
can specify a custom location by setting the `PURPLE_CFG` environmental
variable.

By default, all data will be written to `/data/purpletag`, but you can change
this in the config file.

purpletag fetches the list of legislators and their Twitter handles from
<http://www.govtrack.us/>; these URLs are also specified in the config.


## Getting started

purpletag consists of a number of command-line tools to collect, parse, and
analyze tweets sent by members of Congress.

To see the list of commands:

```
$ purpletag -h
usage: purpletag [--help] <command> [<args>...]

The most commonly used purpletag commands are:
     collect    Collect tweets from members of congress, stored in json
     parse      Parse tweet json
     score      Create score files containing polarization scores for hashtags and MOCs.
     serve      Launch a web service to visualize results.
See 'purpletag help <command>' for more information on a specific command.
```

The expected use case is that `collect` is run continuously, then `parse`, `score`, `serve` are run once daily. There is also support for using historical data (see the `-s` option of `collect` and the `-d` option of `parse`).

### `collect`

This command will fetch tweets from all members of congress listed in `twitter.yaml`.

```
purpletag collect -h
usage:
    purpletag collect [options]
    purpletag collect (-t | --track | -s | --search) [options]

Options
    -h, --help
    -o, --output <file>        output path
    -r, --refresh-handles      fetch latest twitter handles for politicians
    -t, --track                collect tweets in real time using streaming API
    -s, --search               search historical tweets using search API
```

There are two modes of operation:

- `track`: Use the Twitter Streaming API to collect tweets in real-time.
- `search`: Use the Twitter REST API to collect the most recent 3,200 tweets from each legislator.

Output is stored in `/data/purpletag/jsons`.

You probably want to use `search` to first collect all historical tweets, then
run `track` to collect all tweets going forward. **Note:** `search` will take
a long time to run (hours), since the script sleeps to wait out the rate
limits imposed by the REST API.


### `parse`

This command will parse all the collected tweets in `/data/purpletag/jsons`
and extract the hashtags used by each legislator.

```
purpletag parse -h
usage: purpletag parse [options]

Parse .json files into .tags files.

Options
    -h, --help                 help
    -t <timespans>             sliding window timespans [default: 1,7,30]
    -d <days>                  number of historical days to simulate [default: 1]
```

The output looks like this:

```
markwarner whistleblowers:1 studentdebt:1 nova:1 f22:1
repwestmoreland jobs:1 nationaldayofprayer:2 benghazi:3
```

For example, this indicates that Lynn Westmoreland used the hashtag #jobs
once, #nationaldayofprayer twice, and #benghazi three times.

The `-t` parameter indicates a list of timespans to use when aggregating these
statistics. For example `purpletag parse -t 30` will parse all tweets
posted in the past 30 days and compute output like the example above. The
file name itself will indicate this. For example, `2014-05-02.30.tags` is a
tags file created when running this command on May 2, 2014, collecting
statistics for the past 30 days.

The `-d` parameter allows you to simulate running this for a number of days in the past. This is useful after running `purpletag collect -s` to collect all historical data (up to 3,200 per legislator), then generating tags files as if you had been running this daily.

Output is stored in `/data/purpletag/tags`.

### `score`

This command scores hashtags according to their polarity.

```
purpletag score -h
usage: purpletag score [options]

Compute polarity scores for all .tags files that we haven't yet processed.

Options
    -h, --help             help
    -r, --refresh-mocs     fetch latest legislator information from GovTrack
    -c, --counts           use hashtag count features instead of binary features
    -o, --overwrite        overwrite existing .scores files
    -s, --score-mocs	   calculate MOC scores from tag scores
```

These produce `.scores` files and `.moc.scores`, one per `.tags` file. E.g.,
`2014-05-02.365.scores` contains the scores for the hashtags used for the 365
days prior to May 2, 2014. `2014-05-02.365.moc.scores` contains the scores for MOCs who used the hashtags scored in `2014-05-02.365.scores`. The scores range from -Inf (liberal) to +Inf
(conservative). 

The tags' scores are computed by the following approach:

1. Compute the [Chi-squared](http://en.wikipedia.org/wiki/Chi-squared_test) value for each hashtag (i.e., how correlated is it with a party?)
2. Determine the signed version of this score (i.e., is it more indicative of liberals (-) or conservatives (+).
3. Compute the [z-score](http://en.wikipedia.org/wiki/Standard_score) for all tags in this file.

For example:

```
raisethewage -33.2568
1010means -21.485
renewui -17.2857
.
.
.
keystonexl 10.3592
irs 10.9261
obamacare 14.5419
```
The MOC scores are computed by the following approach:

1. Get a list of the tags the MOC used
2. Sum their scores (if the `--counts` option is used, those scores are first multiplied by the number of times the tag was used)

For example:

```
nitalowey -2326.2
senatorbaldwin -1879.52
janschakowsky -1877.39
.
.
.
stevedaines 879.76
senshelby 945.492
speakerboehner 5681.58

```

Output is stored in `/data/purpletag/scores`.

### `serve`

This command will launch a [SimpleHTTPWebServer](https://docs.python.org/2/library/simplehttpserver.html) to visualize tag polarity over time, using [`dygraphs`](http://dygraphs.com/)

```
purpletag serve -h
usage: purpletag serve [options]

Launch a web service to visualize results.

Options
    -h, --help             help
    -n <tags>              number of tags to show from each party [default: 100]
```

The web data is stored in `/data/purpletag/web`. The default port is set by the config file. So <http://0.0.0.0:8000/1.html> might look something like this:

![sample](https://raw.githubusercontent.com/casmlab/purpletag/master/docs/sample-graph.png)
