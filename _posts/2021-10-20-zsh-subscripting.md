---
title: subscripting in zsh
date: 2021-10-21T03:30:30-04:00
categories:
  - blog
tags:
  - zsh
  - scripting
---
Like most shells, zsh support arrays.
```zsh
ary=(foo bar baz)
typeset -p ary
# typeset -a ary=( foo bar baz )
```
With such an array, you can access elements via subscripting like so:
```zsh
echo $ary[1]
# foo
```
Users usually coming from bash or ksh will quickly notice that arrays
in zsh are 1 based, this drums up a sort of debate among software developers
about the proper way indexing should happen. With zero-based indexing being the
most common among programming languages, so clearly zsh is weird in choosing
1-based indexing and to write off zsh's _subscripting_ as lesser than in someway.

This debate is lost upon me being the glue code writing sysadmin that i am,
which prioritizes functionally. Functioningly zsh's _subscripting_ is more
powerful than the shells that i am aware of.

For the curious though, the reason _why_ zsh's indexes start at 1 is most
likely due to one of it's early design goals of appealing to _csh_ users
which was a lot more common at the time. csh, released in 1979 had arrays
seemingly since it's original release, which was 1-based. Making it the
earliest that I am aware of to support arrays.
```zsh
# earliest man page that i can find personally.
man =(curl -sL https://github.com/llua/2bsd/raw/master/2bsd.tar.gz | tar -zOxf- man/csh.u)
```
csh itself most likely being a product of _it's_ time too
since _awk_ (1977-ish), arrays were 1-based
```zsh
awk 'BEGIN { split("foo bar baz", ary); print ary[1] }'
# foo
```
along with shell positional parameters `$1...$n`, which later made up `$@` and
`$*` _(foreshadowing)_.

So, anyway, subscripting. zsh's evalutes the contents of a subscript inside a
arithmetic context, so anything you would usually do inside of `(())` or `$(())`
is possible, the resulting number is indexed.
```zsh
print ${ary[1+1]}
# bar
```
which isn't unique but mentioned for completeness.

Something that is less common among bourne-like shells is specifying a range.
```zsh
print ${ary[2,3]}
# bar baz
ksh93 -c 'ary=(foo bar baz); print "${ary[1..2]}"'
# bar baz
```
If you are mostly a user of bash, you may mention the expansion
`${var:offset:length}` as an alternative. But really it isn't, it is a specifc form
of parameter expansion; which prevents the use of other forms of expansions on a
range of elements.
```zsh
print ${ary[2,3]#ba}
# r z
ksh93 -c 'ary=(foo bar baz); print "${ary[1..2]#ba}"'
# r z
bash -c 'ary=(foo bar baz); for arg in "${ary[@]:1:2}"; do printf "%s\n" "${arg#ba}"; done'
# r
# z
```

zsh, did adopt the `${var:offset:length}` expansion for feature parity but
unlike ksh and bash allows parameter expansions to nest, which makes it doable,
though awkward.
```zsh
print "${(@)${ary[@]:1:2}#ba}"
# r z
```

In addition to specifying ranges, zsh subscripts have _flags_ to change the
meaning how the subscript indexes. (see zshparam(1))

makes subscripting index the words of a scalar parameter
```zsh
str='Lorem ipsum dolor sit amet, consectetur adipiscing elit'
print $str[3]
# r
print $str[(w)3]
# dolor
print $str[(w)3,(w)-1]
# dolor sit amet, consectetur adipiscing elit
```

treat the subscript as a pattern to match against keys, selecting the keys of
the associate array.
```zsh
print $userdirs[(I)l*]
# llua lxdm
```

Or as a pattern matched against values, then using a parameter expansion flag
to get the keys of those values.
```zsh
# prints the username of users with those homedirs
print ${(k)userdirs[(R)/(nonexistent)#]}
# tty games nobody kmem news cyrus proxy pop messagebus bin operator www git_daemon bind
```

Other than ranges and flags, the fact that indexes start at 1 in zsh avoids
weird edge cases like in other shells.
```bash
ary=(foo bar baz); set -- "${ary[@]}"
printf '%s\n' "${ary[*]:0:2}" "${*:0:2}"
# foo bar
# bash foo
```
The second line is probably not what one would expect to happen.
Since when performing expansions with `$@` or `$*` you usually get
```bash
printf '%s\n' "$*"
# foo bar baz # argv[0]/$0 is missing
```
I imagine since positional parameters _starts_ at _1_, `bash` is added to
index 0 to keep positioning consisent in a different, more likely unexpected
way. But it is inconsisent with how one does `"$@"` or `"$*"`.

That isn't the case with zsh's normal way of subscripting.
```zsh
print "$ary[1,2]"
# foo bar
print "$@[1,2]"
# foo bar
print "${ary[@]:0:2}"
# foo bar
# whether or not zsh not being index 0 is a bug depends...
```

So yeah, it is possibly a weird choice to do 1-based indexing, but zsh's
subscripting is still better than most bourne-like shells.
