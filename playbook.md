# Deshell Playbook

Converting shell code to a real language (e.g. C++/Rust).

* Author: Mike Frysinger <vapier@google.com>
* License: [Creative Commons Attribution 4.0](./LICENSE)

## Problem Statement

Shell code in production is a liability.
It is a significant drag on productivity, velocity, and stability.

The larger the body of shell code, the bigger the liability.
Shell code is inherently untestable,
certainly not unit testable ([shUnit] notwithstanding),
a [security nightmare](https://chromium.googlesource.com/chromiumos/docs/+/HEAD/security/noexec_shell_scripts.md)
(being extremely difficult to get right),
requires manual evaluation/QA by developers,
is impossible to automatically validate ([ShellCheck] notwithstanding),
lacks any sort of type checking (everything
[duck types to strings][everything is a string],
is hard to error check (non-zero exit status are ignored by default,
[pipelines make it impossible to check earlier failures](https://mywiki.wooledge.org/BashPitfalls#pipefail),
and [set -e doesn't do what you think it does](https://mywiki.wooledge.org/BashFAQ/105)),
is impossible to make performant
(it naturally forks+execs tons of helper programs, and only in a single thread),
and suffers constantly from underquoting
(especially due to pressure from developers who find "excessive" quoting to be
"ugly").

Any code written in shell is a tacit agreement that "this code is not tested",
and any developers updating it most certainly have not performed complete
regression/tests.
They have done the bare minimum and verified their specific use case seems to
work and that's it.

Further, no one is an expert in the "shell language", and all of the tools one
often invokes from it.
Anyone who claims to be an expert is suffering from the [Dunning–Kruger effect]
and is not to be trusted.

Put simply: shell code is impossible to maintain.
Anyone who claims otherwise is misdirecting because they don't want to maintain
the code either.
They want to get in, make their modification for whatever their requirement
might be, and then run away before it catches on fire
(or rather, someone notices it's on fire).

Anything over ~100 lines is like playing
[MK 200cc with a blue shell](https://www.mariowiki.com/Spiny_Shell_(blue)):
it's a question of "when", not "if", it will all explode, and not only does no
one know the exact time it will blow up, it's extremely likely that it will be
at the worst possible time, and everyone will release a stream of expletives as
a result.
And the person who released the blue shell (i.e. the one who wrote the shell
code in the first place) is nowhere to be found.

## Background

Projects will get to the point where they accept that a particular script has
gotten too large and must be converted for overall code health.
Once that happens, developers usually don't have a great plan for how to
accomplish that goal.
Here we set out a general playbook for anyone to follow:
it is not specific to any language (what the tool will be converted to),
nor any product (CrOS/Android/etc...).

This playbook has been successfully used in CrOS a few times already.
The time frames below are from when the migration started (first change landed
to include the new non-shell program wrapping the old shell script) to when they
finished (the shell script was fully removed).
The fact that some stretch out is actually a good thing and shows that the
process is "working": it means we're comfortable with the stability of the
migration, and we have time to prove that the new bits are stable in the real
world.
Obviously this is a trade-off with "migrations that never finish are bad", but
that leaves the time table up to the maintaining teams to determine their
priority rather than the overall approach forcing a tight schedule with
penalties for slippage.

*   CrOS [crash_sender](https://crbug.com/391887) -
    800 line shell -> C++ over ~10 months and ~7 OS releases
*   CrOS [dev-install](https://crbug.com/935142) -
    300 line shell -> C++ over ~9 months and ~6 OS releases
*   CrOS [setup_board](https://crbug.com/893748) -
    shell -> Python
*   CrOS [crosh](https://crbug.com/979224) -
    2000 line shell -> Rust over 1.5+ years and ongoing
*   CrOS [periodic_scheduler](https://crbug.com/1086719) -
    100 line shell -> C++ over ~1 month
*   CrOS [clobber-state](https://crbug.com/884520) -
    600 line shell -> C++ over ~3 months and ~2 OS releases
*   CrOS board_specific_setup.sh -
    2600 line shell -> Python over ~6 months
*   CrOS [build_packages](https://issuetracker.google.com/187793559) -
    550 line shell -> Python over ~2 months and ongoing
*   CrOS [chromeos_startup](https://issuetracker.google.com//172210918) -
    1300 line shell -> C++ over ~3 years with a few smooth rollbacks
*   CrOS [ARC mount-passthrough-jailed](https://issuetracker.google.com/269713102) -
    130 line shell -> C++
*   CrOS [minidiag](https://issuetracker.google.com/220074034) -
    50 line shell -> C++

There's a few in the progress of starting up:

*   CrOS [chromeos-install](https://issuetracker.google.com/176492189) -
    1200 line shell
*   CrOS [build_image](https://issuetracker.google.com/187786772) -
    2200 line shell -> Python
*   CrOS [swap-init](https://issuetracker.google.com/218519699) -
    700 line shell
*   CrOS cr50-scripts
*   CrOS flash_fp_mcu

## Requirements and scale

Here we set out requirements for the playbook.
It doesn't mean that when you execute the playbook for your own project you
can't adapt it to your needs (and perhaps break some of these rules).

*   The program needs to be converted seamlessly to the rest of the world.
    *   Language implementation is an internal detail.
    *   No one outside of the program authors need to even know it changed.
*   Everything that worked before needs to keep working.
    *   Anything you want to delete/deprecate must be done before or after the
        rewrite, not as part of it.
*   Unittest coverage increases.
    *   It's basically a guarantee that the existing shell code had no coverage.
*   Migration is smooth & continuous.
    *   No disruptive "[flag days]".
    *   Can be worked on in parallel with other tasks, or as time permits.
    *   Can be handed off to other people/teams without throwing away work.
*   Migration is incremental.
    *   Small changes are easier to review, reason about, validate, bisect,
        and revert.
    *   Easier to make course corrections as the migration plays out.
*   Project is not frozen.
    *   People can still add new features or update the existing codebase.
    *   As the migration gets further along, people can add new features to the
        new codebase and avoid touching the shell code entirely.
*   No large rewrites in isolation.
    *   Code not in production is code that can't be trusted.
    *   It's tempting to go down the rabbit hole of cleanups, but those are
        unnecessarily risky.
*   [Rollback]/revert is easy.
    *   Avoid disruptive rollforward when everything is on fire.

## Playbook

The general approach here is to wrap & subsume from the outside in.
The initial replacement will have very little to no code in it, and
functionality will be migrated from the shell code to the replacement
project.
But both will be fully active as long as they exist.

This might mean there is some level of duplication between the two codebases,
but we'll strive to keep it to a minimum, and hopefully only to the point
where we hand off (i.e. we execute the shell script and pass it state).

### Prep Work

Are there accumulated features/functionality that are no longer used?
Now is a great time to deprecate & delete that stuff.
Code that doesn't exist doesn't have to be migrated, so give it a good scrub.

If you accidentally delete something that someone is using, that's a good thing!
This will help you identify those important (but undocumented) use cases that
will be critical to validating the replacement as it's developed.

So keep notes as to what is still needed & where, and write them down where
others can find them.
You might have to hand off migration to someone else, so don't let these early
investigations go to waste.
Post them to issue trackers, or design docs, or wherever else makes sense.

### Wrap It

The goal here is to stand up the infrastructure for building the new code,
as well as any skeleton unittest runners.
Often going from "copy foo.sh to /usr/bin/foo" to "compile foo & install foo &
run foo_unittest" means changing a single line build rule to many rules spread
over multiple files.
And rarely, updating any expectations that runtime or image-fixation-time steps
run that did not expect a compiled program where there was once a shell script.
Shake that all out now without breaking any existing logic.

*   Move the shell script somewhere where developers can't easily access it.
    *   If it was installed at /usr/bin/foo, move it to /usr/libexec/foo.sh.
        Even /usr/bin/foo.sh is OK.
*   Create a simple program that is the entry point.
    *   E.g. For C++ programs, this would be `main()`.
    *   Have it exec the shell script & pass along any command line arguments.
        Don't bother checking the arguments at all at this point.
*   Decide on the source code layout.
    *   For languages that require a `main()` function, you can't normally
        unittest that module directly.
    *   Decide on the filenames to minimize the amount of code in the `main()`
        function.
    *   See example layouts below for more information.
*   Install the new program where the shell script used to be.
    *   It is now /usr/bin/foo.
*   Create a skeleton unittest runner.
    *   Won't really be able to test much at this point, but it'll give build
        rules the right thing to run.
    *   This is where the main-vs-non-main code split helps out: you can link
        the non-main module in.

#### Example C++ layout

*   foo.cc: All module code in class form ready to be unittested.
*   foo.h: Class definitions for foo.cc.
*   foo_main.cc: Includes foo.h, instantiates the class in there, and executes
    it (e.g. calls Run()).

You can decide how command line options get passed down to the class.
Maybe the constructor takes it, or there's a factory method FromCommandLine,
or they're passed to Run, or something else.
But command line option parsing should *not* be in foo_main.cc as even that can
get complicated (e.g. argument type validation, constraint checking, conflicting
options, etc...).

### Peel & Migrate

Now that you have a skeleton, start replacing discrete parts starting from the
outermost layers.
This will require understanding of the shell code that's being replaced, and
identifying sections that could be pulled out independently.

This is the rest of the playbook: identify a chunk, move it over, add unittest
coverage when you do, delete the shell code that's been replaced.
Repeat until the shell script no longer exists.
Celebrate with peer bonuses.

#### Tip #0: Write for testability

The whole point of this migration is to improve test coverage & maintenance
right?
It'd be a shame to do all this work and forget to actually address that
shortcoming, and it'll mean losing confidence in the replacement program as it's
rolled out that it's truly "better".

So as you migrate any code over, remember to write it in such a way that it can
be unittested, and then actually add unittests for it.

It's not uncommon for this process to shake out edge cases that were missed in
the shell code.

#### Tip #1: Usage/help processing

An easy & low risk way to move a decent number of lines to the replacement
program is to migrate --help/usage information.
This will force you to setup basic CLI processing and choose the API in the
process, e.g. [getopt] or some argparse API.
It will also get you comfortable with the migration process.

Then you can delete the content from the shell script and rejoice!
If the shell script was calling usage() to emit extended help information before
exiting, just replace it with a terse message like:

```sh
error: ...
Please run foo --help for more information.
```

This is a common standard in the *NIX world already, so users hopefully
shouldn't be too annoyed by it.

#### Tip #2: CLI processing

Start moving more command line parsing & validation.
Shell scripts often want arguments in a specific format (date/integer),
or have constraints (--foo requires --bar),
or have conflicting requirements (--foo cannot be used with --bar),
or some other thing that should be asserted before the code is run.
Moving the command line options over now also makes it easier to get access
to the state as soon as possible in the new program.

You might have to duplicate some level of option parsing at this point (e.g.
C++ & shell script both accept --foo=<file>), but you should be able to delete
all the validation logic (e.g. C++ asserts <file> exists, but shell script
assumes it exists).
Since the shell script has been removed from direct access to other
programs/developers, it is OK to assume that it is never given bad arguments.

Similarly, you can remove duplicate options from the shell script.
Maybe you have short options or aliases like -f/--foo/--foobar.
Have the replacement program handle them itself and change the shell script to
only accept --foobar.

Depending on the interface, you might even merge some options.
Since the program<->script interface is the only thing that needs to work,
you could make it more terse.
If --output is always required, change it to a positional argument, e.g. the
shell script assumes $1 is the output file.

`foo.sh --output=<file> ...` -> `foo.sh <file> ...`

##### Options Parsers

This tip does not require or assume that either runtime environment is using a
standard library for processing CLI options.
All it requires is that you have a way of ingesting a list of strings (the
command line arguments), process them, and then return the parsed settings.
This provides a black box that can be unittested and where you can enumerate
common and important scenarios that need to keep working.
If your parser breaks those, then you'll probably break users too :).

There are a number of "common" ways to do this in the shell world (e.g.
[POSIX getopts builtin](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/getopts.html),
the [getopt *NIX program](https://man7.org/linux/man-pages/man1/getopt.1.html),
[shFlags](https://github.com/kward/shflags),
or just an ad-hoc loop+case statement) each with their own nuances &
idiosyncrasies.
Ideally your new implementation would leverage an off-the-shelf parser library
so you don't have to implement the messy details yourself, and so you provide a
more standard interface that people would be used to (rather than your custom
funky forms).

Depending on how "bug compatible" you need to be with the existing shell code,
you might have to implement a pre-parser pass that converts old forms to the new
ones and warns about the old deprecated form in the process.
Then add unittest coverage for that pre-parser pass :).

Common nuances that are easy to miss:

*   How are options separated from their values?
    Is it always with spaces (`--option arg`), or with an equals sign
    (`--option=arg`), or are both supported?
*   How are short-vs-long options handled?
    Can long options be specified in the non-standard single dash form like
    `-long-option` or is it always `--long-option`?
    Normally single dashes are used to group multiple short options, so
    `-long-option` is actually shorthand for `-l -o -n -g -- -o -p -t -i -o -n`.
*   How are optional arguments handled with short options?
    Does it always consume another argument, or only when it's in a single flag?
    For example, if `-d` takes an optional number, are both `-d0` and `-d 0`
    allowed?
    What if it's followed by what looks like another option like `-d -v`?
    Is an equals sign supported like `-d=0`?
*   How are optional arguments handled with long options?  See above.
*   How are options-vs-positional arguments handled?
    Does it always require a `--` separator?
    Or does it start processing at the first unknown argument?
    So does `-x foo -b` result in [-x, -b] & [foo], or in [-x] & [foo, -b]?
*   How are boolean options handled?
    For option `--foo`, does the parser allow `--nofoo` or `--no-foo` or
    `--foo=true` or `--foo=false` or `--foo=0` or `--foo=1` or `--foo=y` or
    `--foo=Y` or `--foo=yes` or ...
*   How are integer options handled?
    Does the parser allow for non-base 10 values?
    Does a `0x` prefix indicate a base-16 (hex) number?
    Does a `0` prefix indicate a base-8 (octal) number?
*   Are abbreviated unambiguous long options supported?
    Is `--y` or `--ye` shortcuts to `--yes` even if not explicitly enumerated
    as such?
*   How are dashes & underscores handled?
    Is it always `--long_option`, or always `--long-option`, or are they both
    supported as aliases?

#### Tip #3: Get a handle on the environ

Some shell scripts expect arguments or settings to be passed in via environment
variables.
This is generally a bad interface, and you could/should take the opportunity to
fix that when possible.
Have your replacement program process those env vars (i.e. set internal
variables to them) asap so they can be removed (to avoid leaking to children),
and then either set them back from the internal program state or add CLI options
to the shell script so they can be passed down.

#### Tip #4: Document duplication

Sometimes a utility function is used heavily throughout the shell script, and in
the process of moving a chunk of code, you have to implement the utility
function too, but you can't delete the utility function from the shell code as
it's still used by other places.
Don't fret, document!

In both places, leave a breadcrumb comment in both copies that says "Keep in
sync with <other file>:<other func>.", and perhaps leave a note explaining when
the copy can be deleted entirely.

## Alternatives considered

### Reimplement (in isolation) & migrate on Flag Days

When looking at a large body of untested code, it can be tempting to want to
throw it away and write something from scratch.
The new project will be so beautiful & clean & free of [tech debt] that plagued
the previous project right?
And all the bugs the project had were because of its [tech debt], so your new
project will be bug free?

This is a myth.
*[Kein Operationsplan reicht mit einiger Sicherheit über das erste Zusammentreffen mit der feindlichen Hauptmacht hinaus](https://en.wikiquote.org/wiki/Helmuth_von_Moltke_the_Elder)*.

Attempting to rewrite & replace complex code that has been live in production
for a long time & is battle hardened with something that has never touched the
real world is guaranteed to fail.
It's going to fail the first time it's merged and cause pain for users &
teammates, it's going to fail when rolledback, it's going to fail the second
time it's merged, and it's going to just keep ping-ponging between these
states.
You might eventually force your way through and get the change to stick and
shake out all the regressions, but it's going to be a hot mess the whole time.
More often than not, the project will ultimately be abandoned, and all the
effort will be wasted, and all the user pain for nothing.

Also remember that for active projects, you're always chasing a moving target.
You might have done a lot of comparisons to the project as it existed yesterday
and things look great, but then someone else merged a big new feature today to
it, and that has invalidated a lot of your work forcing you to start again.

[See the FAQ below too for the myth "rewriting takes less time than inplace migrations".][faq2]

Real world examples (of this approach in general, not necessarily specific to
shell -> non-shell migration):

*   [crash_sender](https://crrev.com/c/217740) in CrOS took a long time to
    review the huge single migration, then was reverted due to regressions, and
    then abandoned.  4 years later, this playbook was used successfully.
*   [Uber app rewrite](https://twitter.com/StanTwinB/status/1336890442768547845)
    (ObjC -> Swift) leads to continuous firedrills/escalations for months.
*   (CrOS) [pack_firmware](https://issuetracker.google.com/172339303)
    rewritten in Python in parallel and was a bumpy landing.

### Migrate from the inside-out

Rather than replace the shell script from the outside-in (have the new program
be the main() entry point and keep the shell code as an internal implementation
detail), swap out internal steps of the shell script with the new language.

This approach is a bit limited in that it only works for scripts that are
largely linear and don't have a lot of ambient settings floating around (env
vars, command line options, etc...).
While in shell code it's natural/common to use env vars to hold state, it's the
opposite in real languages as they function as global variables which are
strongly discouraged anti-patterns.

It also complicates the unittesting story as you end up carrying a lot of the
shell code warts over to the new project: the clumsy command line parsing,
[everything is a string], etc...

You might run into processing limitations if your new tool is run in a for loop.
Maybe you've replaced the entire body of the for loop, and now you want to
replace the for loop itself, but it's not always easy to pass that shell state
to the program to have it do the iteration itself.

The outside-in approach proposed in the playbook above has none of these
limitations and focuses on taking robust program state and "lowering" it to
shell code rather than the other way around.

## FAQ

### I can see my teammate holding the blue shell but they haven't released it yet (merged). How can I stop it?

Focus on the project's agreed upon policies:

*   Do you have policies dictating which languages are acceptable & when?
*   Do you have policies requiring test coverage?
*   Do you have style guides that say when shell code is/is not appropriate?

Remind them of those, and that the project has agreed on these, and it isn't
you who is forcing your own personal opinion ("I hate shell") on them.

Remember that the point of these guides isn't to annoy developers, it's for long
term maintenance & health of the project.
Developers do not write code in isolation, and they will not be around at all
times (vacation, sick time, etc...) or generally forever (change teams,
projects, leave the company, "[bus factor]", layoffs, etc...).
So just because they understand well how the code works doesn't mean it's
acceptable to merge.
Someone at some point is going to have to get in there who is not the original
author and be confident that they can make changes without breaking things.

### Inplace migration requires a lot of work.  Rewriting would take less time.  Shouldn't we rewrite?

This thought process is pretty natural: if you don't have to migrate
[tech debt], and only implement the functionality you need from the very
beginning, then wouldn't a rewrite take less time?

The problem here is that it discounts or overlooks the whole story.
Rewriting the code is only the start of the process: you still have to replace
the program in production, and then deal with the long tail of regressions
that the replacement inevitably introduces.
This could take the form of breakage in production (costing users & other
developers), then the reverts & relands, not to mention the time spent tracking
down where in these two large unrelated piles of code the regression came from
and fixing it.

So when you measure the total time & effort required by these two approaches and
not just the "rewrite the code" portion, the process outlined in this document
is cheaper & smoother for everyone involved.

### How to handle moving to a new location without breaking existing users?

This guide focuses a bit on tools that are installed & run from a different
location than where they are maintained as is common in the wider open source
packaging world.
For example, a project locally has a src/foo.sh file, but no one executes that
directly.
Instead it's copied/installed to /usr/bin/foo and users of the tool only ever
execute that.
But what about projects where the exact path is hardcoded (e.g. people are
running ./src/foo.sh directly), and as part of the migration, you want to get
people to run it from a different location, or via $PATH?
As you cut code over from src/foo.sh, you don't want to break existing people,
but you want to inform them of the new method, and you don't want to keep
duplicated code around (which is a bad idea as [documented above][alt1]).
When old paths have been part of your system for a long time, and in your
documentation as The Way Things Are Done, it can be hard to unwind.

The answer is to add another layer!
After you've created the new skeleton tool in the new location, and you rename
the old script to get it away from users, replace the old name with a stub
script that displays a warning message to users before executing the new
location.
This avoids duplication while also informing users of how to migrate, and it
catches code you might have missed in earlier research (e.g. your CI builders).

Here's a more concrete example.  Here is your current project state:

*   old/bin/foo – The old shell script you want to replace, but people are
    running directly
*   new/bin/foo – The new entry point you want people to use
    *   NB: We won't get into the "how do I bootstrap compiled code" topic as
        that's expected to be handled already by your project

So here's the intermediate steps:

*   old/bin/foo.sh – The old/bin/foo script now "hidden" from the user.
    Obviously you could use old/bin/lib/foo.sh or a completely different path.
    The exact name doesn't matter since the point is to just get users to not
    run it directly anymore.
*   new/bin/foo – The new entry point (same as section above).
*   old/bin/foo – A new script you write.
    *   Display a warning message explaining the migration and what users have
        to do.
    *   Execute new/bin/foo with the same arguments.

If you want a form to follow, here's a shell script.
Obviously "how do I handle paths" is something you'll need to adjust accordingly
for your project.

```sh
#!/bin/bash
new_script="new/bin/foo"
cat <<EOF >&2
$0: This script is deprecated and all users must migrate to ${new_script}.
You can simply change all references to $0 to ${new_script}.
This old script will be removed by <some future time>.
If you have questions, please contact <some group>, or file a bug at <issue tracker>.
EOF
exec "${new_script}" "$@"
```


[alt1]: #reimplement-in-isolation--migrate-on-flag-days
[bus factor]: https://en.wikipedia.org/wiki/Bus_factor
[Dunning–Kruger effect]: https://en.wikipedia.org/wiki/Dunning%E2%80%93Kruger_effect
[everything is a string]: https://unix.stackexchange.com/questions/414099/
[faq2]: #inplace-migration-requires-a-lot-of-work--rewriting-would-take-less-time--shouldnt-we-rewrite
[flag days]: https://en.wikipedia.org/wiki/Flag_day_(computing)
[getopt]: https://man7.org/linux/man-pages/man3/getopt.3.html
[Rollback]: https://en.wikipedia.org/wiki/Rollback_(data_management)
[ShellCheck]: https://www.shellcheck.net/
[shUnit]: https://github.com/kward/shunit2
[tech debt]: https://en.wikipedia.org/wiki/Technical_debt
