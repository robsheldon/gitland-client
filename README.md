# My little [gitland](https://github.com/programical/gitland) bot

Gitland was [posted to HN](https://news.ycombinator.com/item?id=22990659) on April 26 and I needed a fun little distraction. I banged out a quick and very sloppy bot and it did pretty well, heavily tilting the board in favor of my team's color.

I had only spent a couple of hours on it, so I posted the bot's code as-was at a2da5c1e44d2b41752e0fa6a0c03d8d7d5e71f20 and decided to retire it the next morning.

And then a couple of opponents suddenly changed tactics and started following other bots, including mine, effectively erasing their moves from the board. Well, I couldn't let that stand. I had guessed early on that that could be an effective strategy, but nobody was doing it so I didn't bother trying to counter it. Now that someone was, the game got a lot more interesting.

I cleaned up and restructured the bot's code and gave it a lot more awareness of the game state and recent activity and then I gave it multiple strategies to use depending on the game state, including one that would punish follower bots.


## State of the code

This is in PHP because that's the language I spend the most time in these days. It doesn't use any of Github's APIs. The code is decent but doesn't follow all of PHP's modern standards. It's a single file with three classes and no dependencies.


## Getting started

You'll need the gitland repo and your gitland-client repo, and a config file that looks something like:
```php
<?php

class config
{
    const GAME_BASE_DIR       = '/path/to/gitland';
    const CLIENT_BASE_DIR     = '/path/to/gitland-client';
    const PLAYER_NAME         = 'yourusername';
    const VALID_MOVES         = ['up', 'down', 'left', 'right', 'idle'];
    const SCORE_OWN_COLOR     = -1;
    const SCORE_OTHER_BOTS    = -4;
    const SCORE_OTHER_COLOR   = 2;
    const SCORE_WEIGHT_FACTOR = 12;
}
```

The config file should be stored in your gitland-client directory. Choose your own values for the `SCORE_*` values. You'll generally want low values for `OWN_COLOR` and `OTHER_BOTS` and a positive value for `OTHER_COLOR`, and a `WEIGHT_FACTOR` higher than 10. Finding optimal values here is pretty much a trial-and-error thing at the moment.

The bot will figure out your team color, current position, and the game state, and then will write its suggested move to the `act` file and try to commit and push the change. Set the bot to run every minute from a crontab and you're done.


## How moves are evaluated

The bot looks across the map in each possible direction from its current position and calculates a total "score" for that direction. It factors in the values in the config file and the current "lifespan" (decay) of each of the tiles. As the calculation moves farther away from the bot's current position, it gradually spreads out, forming a sort of "beam". This helps the bot have a sense of what the board looks like far away from its current position, so it can consistently move towards higher-scoring areas of the board even if the local moves aren't great. Scores are multiplied by the weighting factor, and each additional step away from the bot's current position cuts the weighting factor in half. This helps the bot avoid nearby obstacles, like other bots or its own color, and it handles this very well.

There are a few different strategies that the bot uses depending on the board's situation:

### Escape and Stuck

These two strategies are employed if the bot has limited moves available or finds itself in the middle of a lot of its own color. It first tries to escape, which consists of seeking out the nearest available positive-scoring tile and moving towards it. If it can't escape, then it will switch to "stuck", writing the `idle` command to its `act` file and waiting for the next turn.

### Cruise

If 20% or more of the board is empty (not occupied by any color), then it switches to a cruise strategy. This is a mostly friendly strategy that sends the bot in a positive-scoring direction and keeps it moving in that direction as long as it's still positive. Once the score for that direction becomes negative -- becaause the bot is about to bump into another bot or a lot of its own color -- it will turn. This leaves long, straight lines on the board and plenty of other opportunities for other bots to score points.

### Aggressive

If the board gets more competitive then it switches to a more aggressive strategy and plays the most optimal move each turn. This can carve the board up quite badly and also causes the average age of opponents' tiles to increase.

### Punish

This strategy is my answer to the follower bots. It detects other bots repeating too many of its moves and turns the tables on them, prioritizing their team's color and trying to lead the follower bot into crashing into teammates. It does this by re-scoring the board area and giving positive values to bots on the team that's being punished and very high values to their color tiles. As the bot heads straight into clusters of opponent bots the likelihood that they collide with each other (costing moves and points) goes up. This isn't necessarily the best overall strategy for the bot's team, but it can really make a mess of any team that uses follower bots.


## Walkthrough

Really more of a stumble through the code. Anyway:

### `Map` class

The Map static class at the top of the file contains most of the information about the game state. `Map::load()` does all the file processing in the beginning since most strategies will need all of this data at some point. All player files are loaded so that the Map class can provide information about where other bots are, and a git command is used to load and process recent game activity.

### `Bot` class

The Map class instantiates a list of Bot objects. Each one encapsulates another player's name, team, position, and historical activity. This is a pretty simple class that just provides a clean way to interact with bot-related data.

### `Strategy` class

The Strategy class handles most of the magic decision-making. The application code calls `Strategy::find_move()`, the function responsible for choosing a strategy and returning the chosen move (and commit message).

```php
//  Initialize a default value matrix.
static::$_value_matrix = static::_generate_value_matrix();
```

`find_move()` calls the private function `_generate_value_matrix()`, which calculates the scores from the config file and the decay values for each location in the map. This creates a "base layer" of values for the entire map which can then be combined with the lookahead gradient from the bot's position.

```php
public static function score ($x, $y, $weight = 1)
{
    if ( is_null(Map::get($x, $y)) ) {
        return 0;
    }
    return static::$_value_matrix[$y][$x] * $weight;
}
```

Once the value matrix is calculated, score calculations become fast and easy and can be repeated quickly as the Strategy class works through different options. The `Strategy::area_score()` function does something similar, but just "spreads" the scoring out as it gets farther away from the bot's current position. `Strategy::evaluate_moves()` handles combining the `score()` and `area_score()` results into a single scalar value for each possible direction the bot can move this turn. The `evaluate_moves()` function also includes a fix that prevents bot jitter by penalizing scores in the opposite direction from the bot's last movement. This prevents the bot from oscillating up and down or left and right.

Each of the strategies described above are encapsulated in their own functions: `Strategy::stuck()`, `Strategy::escape()`, `Strategy::cruise()`, `Strategy::aggressive()`, and `Strategy::punish()`. The `Strategy::find_move()` function selects one of these solutions depending on the board situation and the comments in that function pretty well describe the process.


## Possible future improvements

* It should follow standard PHP class-based conventions with composer and all that.
* I would like to optimize the scoring function a bit with some unit testing with atoum or similar. A test suite of different board positions and the expected best move would be great.
* The bot doesn't anticipate movements from other bots. `git log -p log | grep "^\+.*username"` can be used to pick up other bots' movements from the log. This could be used to make a crude guess about what direction a nearby bot is moving and whether a collision is likely.

## That's it.

I had fun. Thanks for setting up the game, @programical. :-)
