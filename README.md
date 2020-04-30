# My little [gitland](https://github.com/programical/gitland) bot

Gitland was [posted to HN](https://news.ycombinator.com/item?id=22990659) on April 26 and I needed a fun little distraction. I banged out a quick and very sloppy bot and it has done very well, heavily tilting the board in favor of my team's color. I'm adding the bot's code and some commentary to the repository before I get tempted to spend any time on it.


## State of the code

It's written in PHP because that's the language I spend the most time in these days. It doesn't use any of Github's APIs. The code is not *good* PHP and doesn't follow many of the modern standards. I've only got a couple hours' time in it at most -- about one hour on the initial versions and another hour fixing edge cases.


## Getting started

You'll need the gitland repo and your gitland-client repo, and a config file that looks something like:
```php
<?php

class config
{
    const GAME_BASE_DIR   = '/path/to/gitland';
    const CLIENT_BASE_DIR = '/path/to/gitland-client';
    const PLAYER_NAME     = 'yourusername';
}
```

The config file should be stored in your gitland-client directory.

The bot will figure out your team color, current position, and the game state, and then will write its suggested move to the `act` file and try to commit and push the change. Set the bot to run every minute from a crontab and you're done.


## How moves are evaluated

There are lots of really really smart algorithms for choosing moves in a game like this. I didn't use any of those. My goal was to see how *quickly* I could throw something together that performed sort of okay. The less time I spent on it, the better.

The bot uses a rudimentary, hacky scoring algorithm to evaluate the board in each of its four possible directions. The scoring algorithm adds positive points for unoccupied enemy colors and negative points for occupied tiles or tiles that already belong to the bot's team. It adds in a weight multiplier for the nearest tiles and cuts the weight multiplier in half as it evaluates each step farther away. This scoring is done in a narrow "beam" in each direction, so the bot can get a sense of what the board looks like far away from its current position.

It looks kind of like this:

```
f    ccc    f
fed  bbb  def
fedcb a bcdef
fedcba#abcdef
fedcb a bcdef
fed  bbb  def
f    ccc    f
```

The "a" positions are really important, the "b" positions are pretty important, "c" positions are less important, and so on.

The scores are added up for each direction and the bot moves in the highest-scoring direction.

This approach works well whether the map is crowded, sparse, mostly full of the bot's color, or mostly full of enemy colors.


## Walkthrough

Really more of a stumble through the code. Anyway:

```php
$moves['left']  = Map::get($current['x'] - 1, $current['y']);
$moves['right'] = Map::get($current['x'] + 1, $current['y']);
$moves['up']    = Map::get($current['x'], $current['y'] - 1);
$moves['down']  = Map::get($current['x'], $current['y'] + 1);
```
After some basic initialization, the bot starts off with figuring out which moves are available from its current position.


```php
$moves = array_filter($moves, function($move){
    if ( is_null($move) || substr($move, 0, 1) == 'c' ) {
        return false;
    }
    return true;
});
```
Invalid moves are removed (moves that go off the board or collide with another bot). If only one move is available, it does that, otherwise:


```php
function evaluate_moves ()
{
    global $moves, $current, $last_move;
    $scores = [];
    foreach ($moves as $direction => $value) {
        $scores[$direction] = 0;
        $x = $current['x'];
        $y = $current['y'];
        $weight = 16;
        $score = 0;
        $spread = 0;
        $lookahead_iteration = 0;
        while ( true ) {
            $lookahead_iteration++;
            //  Expand the lookahead beam a bit.
            if ( $lookahead_iteration % 2 != 0 ) {
                $spread++;
            }
            //  Adjust weighting for distance from current position.
            $weight /= 2;
            //  Transform coordinates.
            list($x, $y) = translate($direction, $x, $y);
            //  Calculate scores in this direction.
            $score = score($x, $y, $weight);
            if ( $score === 0 ) {
                //  score() returns exactly 0 if the coordinates are not valid.
                break;
            }
            //  If this direction is the opposite of the last direction, brutally
            //  penalize it to prevent oscillations.
            if ( $direction == Map::opposite($last_move) ) {
                $score -= $weight;
            }
            //  Calculate the final score for this iteration of this direction.
            $scores[$direction] += ($score + area_score($direction, $x, $y, $weight, $spread));
        }
    }
    arsort($scores);
    return $scores;
}
```
This function does the lookahead scoring for each possible move. The beam's "center" gets a higher weight than the sides of the beam; this allows the bot to find its way through narrow tunnels that open up to more scoring opportunities.

The bot got stuck once when it managed to find a place on the board where "up" was slightly better than down, until it moved one square up, and then "down" looked better, so it moved one square down... ugh. So my last quick fix in this code was a few lines to reduce jitter.


```php
function area_score ($direction, $x, $y, $weight, $spread)
{
    $score = 0;
    $n = -1 * $spread;
    switch ($direction) {
        case 'left':
        case 'right':
            //  If going left or right, then spread vertically.
            while ( $n <= $spread ) {
                $score += score($x, $y + $n, $weight / 2);
                $n++;
            }
            break;
        case 'up':
        case 'down':
            //  If going up or down, then spread horizontally.
            while ( $n <= $spread ) {
                $score += score($x + $n, $y, $weight / 2);
                $n++;
            }
            break;
    }
    return $score;
}
```
This is the function that handles scoring for the expanding "beam" at each iteration.


```php
function score ($x, $y, $weight = 1)
{
    global $current;
    $location = Map::get($x, $y);
    if ( is_null($location) ) {
        return 0;
    }
    if ( substr($location, 0, 1) == 'c' ) {
        //  Strongly avoid other players.
        return -5 * $weight;
    }
    if ( substr($location, 1, 1) != $current['color'] ) {
        //  Try to stomp on other colors.
        return 1.5 * $weight;
    }
    else {
        //  Avoid red areas.
        return -2 * $weight;
    }
    return 0;
}
```
These are all the magic numbers that are used for scoring. -5 points for other bots, 1.5 points for enemy colors, and -2 points for the bot's own color. This is a stupidly simple, fast calculation, and it's good enough to play this game well. The numbers are the result of a few minutes' trial-and-error.


```php
if ( array_key_exists(last_move(), $scores) && $scores[last_move()] > 0 ) {
    echo last_move() . " is still good enough\n";
    $direction = last_move();
}
```
I noticed that the bot was sort of chewing up the board -- zig-zagging all over the place, left, up, left, up, left, up and so on -- and while this was good for my bot, it was bad for all the other players. My fellow teammates would not score points any time they intersected one of my tiles, and opposing teams would have to navigate my paths through their areas or spend a lot of moves on their own color. It wasn't very nice.

So I added this adjustment which would compel the bot to continue in its current direction until it became a bad move. The score evaluation rapidly goes negative as the bot approaches its own area or other bots, so it pivots at just about the right time. This leaves longer lines on the board and more open, square areas for other bots to navigate. If the board becomes very crowded, all directions will become a bit negative, so the bot will return to just choosing the least bad available move each turn.


## Possible improvements

* There shouldn't be globals in modern code. This was a nasty quick hack, but I would remove those and restructure it properly.
* It should follow standard PHP class-based conventions with composer and all that.
* I would like to optimize the scoring function a bit with some unit testing with atoum or similar. A test suite of different board positions and the expected best move would be great.
* There are some edge cases the bot doesn't handle well. Sometimes it will box itself in because it doesn't think far enough ahead about what the board may be like in a few turns. Doing a breadth-first search of the next several moves would be great.
* There isn't a complete model of the board, with usernames for other bots and so on.
* The bot doesn't anticipate movements from other bots. `git log -p log | grep "^\+.*username"` can be used to pick up other bots' movements from the log. This could be used to make a crude guess about what direction a nearby bot is moving and whether a collision is likely.

## That's it.

I had fun. Thanks for setting up the game, @programical. :-)