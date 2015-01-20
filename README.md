## Quick-Start

1. `make setup`
1. `make run`

Your main code is in the `/src/main.coffee`.

## Setup

### BigQueryBot

**BigQueryBot** provides [async](https://github.com/caolan/async) ready functions to control you [Big Query](https://cloud.google.com/bigquery/) execution flow.

Currently supported:

1. `load`
1. `query`
1. `fquery` — same a `query`, but reads it from file
1. `extract`

To setup:

```CoffeeScript
BigQueryBot = (require './big_query_bot.coffee').BigQueryBot

QueryBotConfiguration =
    projectId: '<your project id>'
    datasetId: '<your dataset id>'

bot = new BigQueryBot '<service-account>', '<path-to-pem-file (.p12)>', QueryBotConfiguration
```

### ExtendedBigQueryBot

**ExtendedBigQueryBot** provides [async](https://github.com/caolan/async) ready functions that extend your [Big Query](https://cloud.google.com/bigquery/) execution flow.

Currently supported:

1. `signurl` — provides signed url that is acessible within the next 48 hours
1. `email` — emails results to a specified email through [Mandrill](https://mandrillapp.com)

To setup:

```CoffeeScript

QueryBotConfiguration =
    serviceAccount: '<service-account>'
    privateKeyPath: '<path-to-pem-file (.p12)>'
    projectId: '<your project id>'
    datasetId: '<your dataset id>'

ExtendedBotConfiguration =
    mandrill:
        key: '<your mandrill key>'
        from: '<email to send emails from>'

bot = new ExtendedBigQueryBot QueryBotConfiguration, ExtendedBotConfiguration
```

## Examples

### Sequential

``` CoffeeScript
bot.on 'ready', () ->
    async.waterfall [
        bot.load
            # name:       'import'
            gsPaths:    ['gs://biq-query-bot-sample/*']
            schema:     'line:STRING'

        bot.query
            #name:       'step1'
            sql:        'SELECT * FROM <in> LIMIT 200000'
            #source:     'sniper_by_ip'

        bot.query
            #name:       'step2'
            sql:        'SELECT * FROM <in> LIMIT 100000'
            #overwrite:  true

        bot.query
            # name:       'step3'
            sql:        'SELECT * FROM <in> LIMIT 50000'

        bot.extract ["gs://biq-query-bot-sample/result#{do timestamp}.tsv.gz"]

        do bot.signurl

        bot.graph './graphs/simple.html'

        bot.email 'rubengersons@screen6.io', null

    ], (_, _r) ->
        console.log 'Done'
```

### Parallel

``` CoffeeScript
bot.on 'ready', () ->
    async.parallel [
        bot.load
            gsPaths:    ['gs://biq-query-bot-sample/*']
            schema:     'line:STRING'

        bot.load
            gsPaths:    ['gs://biq-query-bot-sample/*']
            schema:     'line:STRING'

    ], (e, r) ->

        async.waterfall [
            bot.email 'ilya.pimenov@gmail.com', 'sniper logs, imported from gs://', r

            bot.query
                sql:    'SELECT * FROM <in0>, <in1> LIMIT 200000'

            bot.graph './graphs/parallel.html'

            bot.extract ["gs://biq-query-bot-sample/result#{do timestamp}.tsv.gz"]

            do bot.signurl

        ], (_, _r) ->
            console.log "Done, result url #{_r}"
```

### Sniper ETL

``` CoffeeScript
bot.on 'ready', () ->
    async.waterfall [
        bot.load
            gsPaths: ['gs://biq-query-bot-sample/*']
            schema: 'line:STRING'

        bot.query
            file: './src/queries/etl/sniper/01-read-lines.sql'

        bot.query
            file: './src/queries/etl/sniper/02-add-geo-meta.sql'

        bot.query
            file: './src/queries/etl/sniper/03-join-with-geo.sql'

        bot.graph './graphs/etl.html'

        bot.email 'ilya.pimenov@gmail.com', 'sniper etl data for period 10-11.2014 ...'

    ], (_, _r) ->
        console.log 'Done'
```
