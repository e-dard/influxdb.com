---
title: Live Leaderboard of game scores
---

```javascript
// Define a result that contains the top 5 scores by game updated every 1 second.
var topScores = stream
    .from('scores')
    // Get the most recent score for each player
    .groupBy('game', 'player')
    .window()
        .period(1s)
        .every(1s)
        .align()
    .mapReduce(influxql.last('value'))
    // Calculate the top 5 scores per game
    .groupBy('game')
    .mapReduce(influxql.top(5, 'last', 'player'))

// Expose those scores over the HTTP API at the 'top_scores' endpoint.
// Now your app can just request the top scores from Kapacitor
// and always get the most recent result.
topScores
   .httpOut('top_scores')

// Sample the top scores and keep a score once every 10s
var topScoresSampled = topScores.sample(10s)

// Store top five player scores
topScoresSampled
    .influxDBOut()
        .database('game')
        .measurement('top_scores')

// Calculate the max and min top scores and their difference
// and store them back in InfluxDB for later analysis.
var max = topScoresSampled
    .mapReduce(influxql.max('top'))
var min = topScoresSampled
    .mapReduce(influxql.min('top'))

max.join(min)
        .as('max', 'min')
    .eval(lambda: "max.max" - "min.min", lambda: "max.max", lambda: "min.min")
        .as('gap', 'topFirst', 'topLast')
    .influxDBOut()
        .database('game')
        .measurement('top_scores_gap')

```


