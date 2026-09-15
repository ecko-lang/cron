# Cron - Ecko Std Lib Package

Parse a cron expression, ask when it next fires, and test whether a given
moment matches. Pairs with `std.bg` and long-running `std.http` servers.

## Install

```bash
ecko get github.com/ecko-lang/cron
```

```ecko
import cron
```

Pure - no capabilities.

## Usage

```ecko
import cron
import std.time

s = cron.parse("*/15 9-17 * * mon-fri")

cron.next(s, time.now())      # the next quarter-hour in office hours
cron.matches(s, time.now())   # is right now one of them?
```

A scheduler loop is the usual shape:

```ecko
schedule = cron.parse("0 3 * * *")
mut due = cron.next(schedule, time.now())

loop {
    if time.now() >= due {
        digest_overnight_mail()
        due = cron.next(schedule, time.now())
    }
    sleep(30)
}
```

## The five fields

```
 minute  hour  day-of-month  month  day-of-week
 0-59    0-23  1-31          1-12   0-6 (0 = Sunday)
```

Each accepts `*`, a number, a range `a-b`, a step `*/n` or `a-b/n`, a bare
number with a step (`5/10` means from 5, every 10), and a comma-separated list
of any of those.

Months and weekdays may be named and are case-insensitive: `jan`, `JAN`,
`jan-mar`, `mon-fri`. `7` is accepted for Sunday, as elsewhere.

Shortcuts: `@yearly`, `@annually`, `@monthly`, `@weekly`, `@daily`,
`@midnight`, `@hourly`.

## Two behaviours worth knowing

**Day-of-month and day-of-week together mean EITHER.** This surprises people,
and it is the documented POSIX rule rather than a quirk here. When both fields
are restricted, a day matching *either* fires:

```ecko
cron.parse("0 0 15 * mon")   # the 15th, AND every Monday
```

Restricting only one of them is a plain AND with the rest of the expression.

**Everything is UTC.** A scheduler that silently used local time would fire an
hour out twice a year and be right the rest of the time, which is the worst kind
of wrong. Convert before you call if you want local time.

## Impossible schedules

`next` searches four years ahead, which covers every leap cycle, then gives up
with an error of kind `cron`:

```ecko
cron.next(cron.parse("0 0 30 2 *"), time.now())
# cron: '0 0 30 2 *' has no next firing time - check the day and month
```

The 30th of February never arrives. Answering `null` would make a scheduler loop
spin silently; raising says what is wrong.

`0 0 29 2 *` is not impossible - it finds the next leap year, skipping up to
three.

## API

| call | what it does |
|---|---|
| `parse(expr)` | A schedule, or raises kind `cron` naming the field at fault. |
| `next(schedule, from_ms)` | The first firing time strictly after `from_ms`, in Unix milliseconds. |
| `matches(schedule, ms)` | Whether the minute containing `ms` is one it fires in. |

## Notes

Cron's resolution is one minute, so `matches` ignores seconds and `next` never
answers a time in the past because of them.

The calendar arithmetic is self-contained rather than built on the `datetime`
package. Not preference: a package that declares a dependency cannot ship
correctly today, because the release workflow archives `ecko.json` and the root
`*.ecko` files only, leaving a vendored dependency out of the zip.

## Testing

```bash
ecko test tests/
```

## License

MIT
