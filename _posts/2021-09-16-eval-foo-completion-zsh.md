---
title: eval "$(cmd completion zsh)"
date: 2021-09-16T03:30:30-04:00
categories:
  - blog
tags:
  - zsh
  - completion
---
This post is intended to point out flaws with the increasingly more common
practice of adding completers (functions used to offer a user possible
choices to select) to shells, with a focus on `zsh`.

New software projects have started to leverage cli libraries that have the
feature of generating a completer for the application supporting common shells,
usually `bash` and zsh are in that list. To utilize these completers, users are
told add something like the following to their shell startup files.

```zsh
source <(cmd completion zsh)
# or
eval "$(cmd completion zsh)"
```

The program prints the completer, and possibly additional code to set the
completer up to standard output and you execute the code. You now have shell
completion for your command, everything is ok in the world. Ignoring the fact
that these completers are written for bash and are shoehorned into zsh's
completion system, making the utilization of it's features more annoying, but
i digress.

Unfortunately one of the problems with this paradigm is the requirement of
executing of the command. Depending on what language it is written in, maybe
Go, but just as common, Python and others; The performance impact on the start
up of their shell can be very noticeable. With the problem only increasing when
they have multiple commands that require this hack, all for completion of
commands that the user may not use in a given session. It is a infrequent
complaint that users have when seeking help in libera’s `#zsh` channel.

How can this be avoided? luckily zsh has _autoloadable functions_. Originally from
`ksh93`, there is a list of directories that zsh can check for shell functions
that when a function is called for the first time, zsh will load the function
into memory and execute it. This list is stored in the array `fpath`, (and tied
to the scalar `FPATH`)

```zsh
% typeset -p fpath
typeset -aUT FPATH fpath=(
/home/llua/.config/functions/wrappers /home/llua/.config/functions-local
/home/llua/.config/functions /usr/share/zsh/site-functions
/usr/share/zsh/functions/Calendar /usr/share/zsh/functions/Chpwd
/usr/share/zsh/functions/Completion /usr/share/zsh/functions/Completion/Base
/usr/share/zsh/functions/Completion/Linux /usr/share/zsh/functions/Completion/Unix
/usr/share/zsh/functions/Completion/X /usr/share/zsh/functions/Completion/Zsh
/usr/share/zsh/functions/Exceptions /usr/share/zsh/functions/Math
/usr/share/zsh/functions/MIME /usr/share/zsh/functions/Misc
/usr/share/zsh/functions/Newuser /usr/share/zsh/functions/Prompts
/usr/share/zsh/functions/TCP /usr/share/zsh/functions/VCS_Info
/usr/share/zsh/functions/VCS_Info/Backends /usr/share/zsh/functions/Zftp
/usr/share/zsh/functions/Zle
)
```

like `path`/`PATH`, you can add to this list as you please.
Afterwards you can mark a function for `autoload` with the command:
```zsh
autoload -Uz myfunction1 myfunction2 [...]
```

zsh’s completion system actually utilizes autoloadable functions heavily. Upon
autoload and running the function `compinit` (to bootstrap the completion
system), compinit marks every function in fpath that begins with a leading `_`
(underscore). These files are usually completers, utility functions, or parts
of the completion system itself. That convention is why you may have noticed a
bunch of weird functions names when completing the names of commands.

The project `bash-completion`, a project separate from bash itself actually
implements autoloadable functions via shell functions (meta and a nice feat) to
make up for the fact that bash itself doesn’t have it currently. This again
offers the perk of not having hundreds of shell functions loaded into ram, to
most likely not be used ever.

With these new commands that generate their own completers, the ideal
recommendation should be to create a file in fpath for zsh or the directory
that bash-completion uses in the program respective install step (make install,
etc). Alternately, people that package the software could do so. But the current
recommended practice of telling users to eval the output of the command really
should be avoided. Leaving it to some niche situation(s) that may call for doing
it interactively but not inside an user’s start up files, it really just does
not scale well for the possible benefit of the session. In addition to the
previous point, more often than not, the only time the output that is being
`eval`'d changes is when the program is updated. Which makes it not very DRY
(don't repeat yourself) from the point of view of the compsys.

With how the libraries (that I have personally seen so far) work right now, you
as a user can’t just replace:
```zsh
source <(foo completion zsh)
# with
foo completion zsh > /path/to/fpath/dir/_foo
```
To speed things up since in addition to the function definition there is a bit
of code (usually at the end). That code associates the completer with the
command that it will handle with `compdef`/`_bash_complete` (or `complete` for
bash) that should be removed. Also a magic comment at the top of the file
`#compdef cmd` is expected and may not be in the output produced by the
completer generator. It is a bit involved to work around the issue and it has
to be done whenever the program updates if it adds new options to the output.

But you can workaround that by using… *drumroll* an _autoloadable function_ to
bootstrap the completer. An example utility can be found here:
[\_cli\_generic](https://github.com/llua/_cli_generic) and it basically works
by associating the command that normally requires doing the eval _trick_ to
`_cli_generic` with compdef:
```zsh
compdef _cli_generic kubectl
```
and once you actually attempt to perform completion for `kubectl`, only then
does the `eval "$(kubectl completion zsh)"` happens, the completer is
executed and eventually matches appear to select from in zsh.
