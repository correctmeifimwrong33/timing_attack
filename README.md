[![Black Hat Arsenal]( https://github.com/toolswatch/badges/blob/master/arsenal/usa/2017.svg)](https://www.toolswatch.org/2017/06/the-black-hat-arsenal-usa-2017-phenomenal-line-up-announced/)
[![Gem Version](https://badge.fury.io/rb/timing_attack.svg)](http://badge.fury.io/rb/timing_attack)

# timing_attack

Profile web applications, sorting inputs into two categories based on
discrepancies in the application's response time.

If you need a known-vulnerable application for testing and/or development, see
[Camelflage](https://github.com/ffleming/camelflage).

## Setup

```bash
$ sudo apt install ruby-{dev,progressbar,json}
$ gem install typhoeus
$ ./timing_attack
```

## Usage

`./timing_attack -u https://example.com/ -c 1 -n 1000 --brute-force --headers '{"X-Api-Key": "INPUT"}'`

```
timing_attack [options] -u <target> <inputs>
    -u, --url URL                    URL of endpoint to profile.  'INPUT' will be replaced with the attack string
    -n, --number NUM                 Requests per input (default: 50)
    -c, --concurrency NUM            Number of concurrent requests (default: 15)
    -t, --threshold NUM              Minimum threshold, in seconds, for meaningfulness (default: 0.025)
    -p, --post                       Use POST, not GET
    -q, --quiet                      Quiet mode (don't display progress bars)
    -b, --brute-force                Brute force mode
    -i, --inputs-file FILE           Read inputs from specified file, one per line
        --parameters STR             JSON hash of URL parameters.  'INPUT' will be replaced with the attack string
        --parameters-file FILE       Name of file containing parameters as with --parameters
        --headers STR                JSON hash of headers.  'INPUT' will be replaced with the attack string
        --headers-file FILE          Name of file containing headers as with --headers
        --body STR                   JSON hash of parameters to be included in the request body.  'INPUT' will be replaced with the attack string
        --body-file FILE             Name of file containing parameters as with --body
        --http-username STR          HTTP basic authentication username.  'INPUT' will be replaced with the attack string
        --http-password STR          HTTP basic authentication password.  'INPUT' will be replaced with the attack string
        --percentile NUM             Use NUMth percentile for calculations (default: 3)
        --mean                       Use mean for calculations
        --median                     Use median for calculations
    -v, --version                    Print version information
    -h, --help                       Display this screen
```

Note that setting concurrency too high can add significant jitter to your results.  If you know that your inputs contain elements in both long and short response groups but your results are bogus, try backing off on concurrency.  The default value of 15 is a good starting place for robust remote targets, but you might need to dial it back to as far as 1 (especially if you're attacking a single-threaded server)

For the `url`, `body`, `headers`, `--http-password`, `http-username`, and
`parameters` options, the string `INPUT` can be included.  It will be replaced
with the current test string.

The `body`, `headers`, and `parameters` options take objects serialized with
JSON.

### Enumeration

Consider that we we want to gather information from a Rails server running
locally at `http://localhost:3000`.  Let's say that we know the following:
* `charles@poodles.com` exists in the database
* `invalid@fake.fake` does not exist in the database

and we want to know if `bactrian@dev.null` and `alpaca@dev.null` exist in
the database.

We execute (using `-q` to suppress the progress bar)
```bash
% timing_attack -q -u 'http://localhost:3000/timing/conditional_hashing?login=INPUT&password=123' \
                bactrian@dev.null alpaca@dev.null \
                charles@poodles.com invalid@fake.fake
```
```
Short tests:
  invalid@fake.fake             0.0031
  alpaca@dev.null               0.0033
Long tests:
  bactrian@dev.null             0.1037
  charles@poodles.com           0.1040
```

Note that you don't need to know anything about the database when attacking.  It
is, however, nice to have a bit of information as a sanity check.

### Brute Forcing

Consider that we know the endpoint
`http://localhost:3000/timing/string_comparison` is vulnerable to a timing
attack due to an early return in string comparison.  We can attack it with
```bash
timing_attack -u http://localhost:3000/timing/string_comparison \
              --parameters '{"password":"INPUT"}' \
              --brute-force
```
This will attempt a brute-force timing attack against against the `password`
parameter.

### Specifying inputs
The URL itself (`--url`), URL parameters (`--parameters`), HTTP body
(`--body`), and HTTP headers (`--headers`) can all contain the string `INPUT`.
`INPUT` will be replaced with the current attack string, whether it is
specified on the command line (as in enumeration mode), or generated by
timing_attack (as in brute force mode).

To perform a timing attack against HTTP basic authentication, `--http-username`
and `--http-password` can be specified.  `INPUT` will be replaced with the
current attack string as above.

The `--parameters` and `--body` options must be specified in JSON format.

## Reading from files

Body contents, parameters, headers, and inputs can all be read from a file
specified on the comamnd line with `--body-file`, `--parameters-file`,
`--headers-file`, and `--inputs-file` respectively.  `--body-file`,
`--parameters-file`, and `--headers-file` expect the file's contents to be a
JSON hash; `--inputs-file` simply expects one input per line.

Example:
```
% cat inputs.txt
charles@poodles.com
camel@sahara.com
woofer@beagles.net
bactrian@dev.null
dromedary@dev.null
alpaca@theand.es
```
```
% cat params.txt
{"login":"INPUT", "password":"123", "delta":"10"}
```
```
% timing_attack -q -u "http://localhost:3000/timing/login" \
                      --parameters-file params.txt \
                      --inputs-file inputs.txt
Short tests:
  woofer@beagles.net            0.0023
  alpaca@theand.es              0.0025
Long tests:
  bactrian@dev.null             0.1042
  charles@poodles.com           0.1046
  camel@sahara.com              0.1051
  dromedary@dev.null            0.1054
```

## How it works

The various inputs are each thrown at the endpoint `--number` times.  The
`--percentile`th percentile of each input's results is considered the
representative result for that input.  Inputs are then sorted according to
their representative results and the largest spike in their graph is found.
Results are then split into short and long groups according to this spike.

The `--mean` flag uses the average of results for a particular input as its
representative result.  The `--median` flag simply uses the 50th percentile.
According to [Crosby, Wallach, and
Reidi](https://www.cs.rice.edu/~dwallach/pub/crosby-timing2009.pdf), results
with percentiles above ~15, median, and mean are all quite noisy, so you should
probably keep `--percentile` low.

I was very surprised to find that I get correct results against remote targets
with `--number` around 20.  Default is 50, as that has been sufficient in my tests
for LAN and local targets.

## Contributing

Bug reports and pull requests are welcome [here](https://github.com/ffleming/timing_attack).

## Disclaimer

timing_attack is quick and dirty.

Also, don't use timing_attack against machines that aren't yours.
