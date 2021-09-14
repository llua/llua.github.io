---
title: display strings and \_pids
date: 2021-09-14T19:42:30-04:00
categories:
  - blog
tags:
  - zsh
  - completion
---

Zsh's completion system allows for you to display arbitrary display strings for
items being completed.

a basic example:
```zsh
_foo() {
  # you usually shouldn't use _describe with this literal
  # array syntax.
  _describe "abc's" '(a b c)' '(1 2 3)'
}
compdef _foo foo
```
In the example above when performing completion for the command `foo`,
zsh will use the _display strings_ `a b c` for the _match strings_ `1 2 3`;
which are possible choices to insert into the command line.

If you've ever seen zsh's completion _descriptions_, then you were looking at
display strings.
```
% ls --color=
 --color  :: control use of color
 -,       :: print file sizes grouped and separated by thousands
 -1       :: single column output
 -A       :: list all except . and ..
 -a       :: list entries starting with .
 -b       :: as -B, but use C escape codes whenever possible
 -B       :: print octal escapes for control characters
 -C       :: list entries in columns sorted vertically
 -c       :: status change time
...
```
Those _descriptions_ are templated display strings made up of the _match string_,
possible padding, a seperator and a string of text describing the _match string_.
The various utility functions for the completion system abstracts that away from
the author of a completer.

Even _\_describe_ can do so:
```zsh
_foo() {
  local -a values=('1:ah... one' '2:ah two...' '3:ah three.')
  _describe 'my tag description' values
}
compdef _foo foo
---
% foo 1
Completing: my tag description
 1  :: ah... one
 2  :: ah two...
 3  :: ah three
```

So on to *\_pids*, when a completer completes process IDs using \_pids you will
see zsh do something pretty cool.
```zsh
% kill 683090
Completing: process ID
683090 pts/7    00:00:00 zsh
683124 pts/7    00:00:00 zsh
683127 pts/7    00:00:00 ps
```
\_pids uses the command `ps` to get available processes and uses the output
as the display strings for the process id match strings. I never knew how it
worked, was curious and poked around to figure it out.

A long time user may know that you can customize that command used since `ps`
without any additional options usually only display processes running in the
same TTY and EUID as the user.
```zsh
zstyle ':completion:*:processes' command \
  'ps -o pid,user,lstart,pcpu,pmem,rss,args -A'
zstyle ':completion:*:processes' format \
  'Completing: %d (pid user lstart %%%cpu %%%mem rss args)'
---
% kill 100
 Completing:  process ID (pid user lstart %cpu %mem rss args)
    100 root     Fri Sep 10 16:14:41 2021  0.0  0.0     0 [mld]
    101 root     Fri Sep 10 16:14:41 2021  0.0  0.0     0 [ipv6_addrconf]
    102 root     Fri Sep 10 16:14:41 2021  0.0  0.0     0 [kworker/0:1H-kblockd]
     10 root     Fri Sep 10 16:14:41 2021  0.0  0.0     0 [rcu_tasks_kthre]
    114 root     Fri Sep 10 16:14:42 2021  0.0  0.0     0 [kstrp]
    119 root     Fri Sep 10 16:14:42 2021  0.0  0.0     0 [zswap1]
...
```

So how would we mimic what \_pids did here? well lets look at how \_pids works.
```zsh
#compdef pflags pcred pldd psig pstack pfiles pwdx pstop prun pwait

# If given the `-m <pattern>' option, this tries to complete only pids
# of processes whose command line match the `<pattern>'.

local out pids list expl match desc listargs all nm ret=1

_tags processes || return 1

if [[ "$1" = -m ]]; then
  all=()
  match="(*[[:blank:]]|)${PREFIX}[0-9]#${SUFFIX}[[:blank:]]*(/|[[:blank:]]-(#c,1))${2}([[:blank:]]*|)"
  shift 2
elif [[ "$PREFIX$SUFFIX" = ([%-]*|[0-9]#) ]]; then
  all=()
  match="(*[[:blank:]]|)${PREFIX}[0-9]#${SUFFIX}[[:blank:]]*"
else
  all=(-P "$IPREFIX" -S "$ISUFFIX" -U)
  match="*[[:blank:]]*[[/[:blank:]]$PREFIX*$SUFFIX*"
  nm="$compstate[nmatches]"
fi

```
So the sake of staying on topic I am going to skip the `#compdef` magic comment
and the comment about `-m` details what the first branch of the `if` statement
is doing. `_tags` is a function that handles valid tags in a given completer;
`tags` being a namespace for _match strings_. The match strings we add here will
be added to the `processes` `tag`/namespace. Largely unimportant right now but
i still wanted to explain it.

The second conditional is checking if the word that is currently attempted to be
completed, if any, matches the various ways a pid is normally given on a command
line. `%jobnumber`, `-processgroupid`, or `processid`. If so, gives a pattern
that will eventually be used to parse the output of `ps`. This is largely the
important part.

The last conditional does something pretty clever that I was unaware of but won't
explain here. But the TL;DR is, it helps implement the `insert-ids` style.

So the next chunk of code is:
```zsh
while _tags; do
  if _requested processes; then
    while _next_label processes expl 'process ID'; do
      out=( "${(@f)$(_call_program $curtag ps 2>/dev/null)}" )
      desc="$out[1]"
      out=( "${(@M)out[2,-1]:#${~match}}" )

      if [[ "$desc" = (#i)(|*[[:blank:]])pid(|[[:blank:]]*) ]]; then
        pids=( "${(@)${(@M)out#${(l.${#desc[1,(r)(#i)[[:blank:]]pid]}..?.)~:-}[^[:blank:]]#}##*[[:blank:]]}" )
      else
        pids=( "${(@)${(@M)out##[^0-9]#[0-9]#}##*[[:blank:]]}" )
      fi
...
```
The first three lines are related to tag stuff but the fourth line is what
allowed us to change the `ps` command used to get _match strings_ via `zstyle`.
When you have a command that a user may want to change, you use `_call_program`
to do so.

`desc=$out[1]` saves the header that ps prints. So for \_pids to work reliably,
if you change the ps command used, try to ensure you do not ommit the headers.

`out=( "${(@M)out[2,-1]:#${~match}}" )` parses the rest of the output, saving
only the lines that matches the pattern saved in the first snippet.

`if [[ "$desc" = (#i)(|*[[:blank:]])pid(|[[:blank:]]*) ]]` this line is
checking if `$desc` is indeed the header line and that it contains `PID`.

If so, it does this beastly of a parameter expansion.
```zsh
pids=( "${(@)${(@M)out#${(l.${#desc[1,(r)(#i)[[:blank:]]pid]}..?.)~:-}[^[:blank:]]#}##*[[:blank:]]}" )
```
That parses out the pids based on the location of the PID column in the output.

Should `$desc` not contain `PID` though
```zsh
pids=( "${(@)${(@M)out##[^0-9]#[0-9]#}##*[[:blank:]]}" )
```
fallback and just parse out any words in the output that are all numbers.

At this point `$pids` contain the match strings while `$out` contain our display
strings (the latter will be saved in another array). The code below checks the 
`verbose` style, which is the idiomatic way of determining whether or not to use
display strings (display _descriptions_) when using low level completion tools
(e.g: not a helper function like \_arguments, \_describe, etc) and if so add
the necessary arguments to pass to `compadd`.
```zsh
compadd "$@" "$expl[@]" "$desc[@]" "$all[@]" -a pids && ret=0
```

So now that we understand how \_pids work, how can we do it with something we
care about?

For this post I will complete OCI image IDs and use `podman` to do so.
```zsh
% podman images
REPOSITORY                                 TAG         IMAGE ID      CREATED        SIZE
docker.io/crystallang/crystal              latest      e7dac73e9072  7 weeks ago    522 MB
localhost/findimagedupes                   latest      fbfc841e5055  3 months ago   399 MB
...
```
podman presents them as a 12 character hexadecimal number so my function will do the same.

I will also use \_pids as a base for my \_podman\_images.
```zsh
#compdef foo

local out images list expl match desc listargs all nm fmt ret=1

_tags images || return 1

if [[ "$PREFIX$SUFFIX" = [[:xdigit:]]# ]]; then
  all=()
  match="(*[[:blank:]]|)${PREFIX}[[:xdigit:]]#${SUFFIX}[[:blank:]]*"
else
  all=(-P "$IPREFIX" -S "$ISUFFIX" -U)
  match="*/${PREFIX}[^[:space:]]$SUFFIX*"
  nm="$compstate[nmatches]"
fi

# allow an easier way to change the display string, versus specifying an entire command
# including this template.
if ! zstyle -s ":completion:${curcontext}:images" template fmt; then
  fmt='table {% raw %}{{.Repository}} {{.Tag}} {{.ID}} {{.Created}} {{.Size}}{% endraw %}'
fi

while _tags; do
  if _requested images; then
    while _next_label images expl 'image ID'; do
      # programs and their arguments given to _call_program are eval'd, so ${(q-)...} preserves quoting
      out=( "${(@f)$(_call_program $curtag podman images --format ${(q-)fmt} 2>/dev/null)}" )
      if [[ $out[1] = *(#i)'image id'* ]]; then
        desc="$out[1]"; shift out
      fi
      out=( "${(@M)out:#${~match}}" )

      # if the first line of output is the header, which what the default _call_program has
      if [[ "$desc" = (#i)(|*[[:blank:]])'IMAGE ID'(|[[:blank:]]*) ]]; then
        # uses the length of the header, + 4 (since the fields are left adjusted) to trim anything after the IMAGE ID column
        # then trim anything before the IMAGE ID column
        images=( "${(@)${(@M)out#${(l.${#desc[1,(r)(#i)[[:blank:]]image id]}+4..?.)~:-}[^[:blank:]]#}##*[[:blank:]]}" )
      else
        images=( "${(@)${(@M)out#*[[:space:]]#[[:xdigit:]](#c12)}##*[[:blank:]]}" )
      fi

      if zstyle -T ":completion:${curcontext}:$curtag" verbose; then
        list=( "${(@Mr:COLUMNS-1:)out}" )
        desc=(-ld list)
      else
        desc=()
      fi
      compadd "$@" "$expl[@]" "$desc[@]" "$all[@]" -a images && ret=0
    done
  fi
  (( ret )) || break
done

if [[ -n "$all" ]]; then
  zstyle -s ":completion:${curcontext}:images" insert-ids out || out=menu

  case "$out" in
  menu)   compstate[insert]=menu ;;
  single) [[ $compstate[nmatches] -ne nm+1 && $compstate[insert] != menu ]] &&
              compstate[insert]= ;;
  *)      [[ ${#:-$PREFIX$SUFFIX} -gt ${#compstate[unambiguous]} ]] &&
              compstate[insert]=menu ;;
  esac
fi

return ret
```

The important changes are documented and the function will achieve what \_pids
does.
```zsh
% foo 7f6f3d95821b
 Completing:  image ID
docker.io/crystallang/crystal              latest      e7dac73e9072  7 weeks ago    522 MB
docker.io/library/archlinux                base        3de742be9254  5 months ago   416 MB
docker.io/library/archlinux                base-devel  7f418864de94  5 months ago   730 MB
docker.io/library/php                      fpm         7f6f3d95821b  5 months ago   415 MB
docker.io/library/ubuntu                   latest      f643c72bc252  9 months ago   75.3 MB
```

This alone may not be the most desired thing to have complete say,
`podman image rm` since in addition to image ids, it can also accept
repository:tag as argument. So a user may be used to using
`podman image rm docker.io/library/archlinux` not
`podman image rm 3de742be9254` to delete images.

This is where the `else` clause of the if condition in \_pids comes into play.
```zsh
...
else
  all=(-P "$IPREFIX" -S "$ISUFFIX" -U)
  match="*[[:blank:]]*[[/[:blank:]]$PREFIX*$SUFFIX*"
  nm="$compstate[nmatches]"
fi
...
```

If the user were to type in an alphanumeric string before attempting to
complete an process id, \_pids will use that string to match against the
output of `ps` and for any line that the string matches, present the
pid of the line as a choice in the match strings.

```zsh
% kill firefox<tab>
---
% kill 447001
 Completing:  process ID (pid user lstart %cpu %mem rss args)
 447001 llua     Mon Sep 13 09:41:02 2021  0.0  1.1 180872 /usr/lib/firefox/firefox -cont
 447032 llua     Mon Sep 13 09:41:03 2021  0.1  1.4 230308 /usr/lib/firefox/firefox -cont
   4866 llua     Fri Sep 10 16:16:03 2021  3.9  4.2 680364 /usr/lib/firefox/firefox
   4934 llua     Fri Sep 10 16:16:04 2021  0.7  1.5 250652 /usr/lib/firefox/firefox -cont
   4970 llua     Fri Sep 10 16:16:05 2021  0.4  1.7 286452 /usr/lib/firefox/firefox -cont
   4974 llua     Fri Sep 10 16:16:05 2021  0.5  2.7 439960 /usr/lib/firefox/firefox -cont
   4978 llua     Fri Sep 10 16:16:05 2021  0.3  3.6 585772 /usr/lib/firefox/firefox -cont
   4982 llua     Fri Sep 10 16:16:05 2021  2.4  2.2 368488 /usr/lib/firefox/firefox -cont
   5049 llua     Fri Sep 10 16:16:05 2021  0.5  0.6 101100 /usr/lib/firefox/firefox -cont
   5113 llua     Fri Sep 10 16:16:05 2021  1.4  2.7 445104 /usr/lib/firefox/firefox -cont
   5384 llua     Fri Sep 10 16:16:11 2021  0.6  0.4 71556 /usr/lib/firefox/firefox -conte
   8701 llua     Fri Sep 10 16:32:48 2021  0.2  1.3 211284 /usr/lib/firefox/firefox -cont
```

With our copycat doing the same
```zsh
% foo archl<tab>
---
% foo 3de742be9254
 Completing:  image ID
docker.io/library/archlinux                base        3de742be9254  5 months ago   416 MB
docker.io/library/archlinux                base-devel  7f418864de94  5 months ago   730 MB
localhost/archlinux-tools                  latest      56a21332be3d  5 months ago   848 MB
```

Ultimately, using the output of a command as a display string it is a creative
way of using menu selection. Something that is as far as i am aware, uniquely
possible in zsh. At the same time, now that i know _how_ to do it, I am still
not eager make as much of my personal completers do the same since i can
easily see this become a pain to maintain for more complicated programs.
