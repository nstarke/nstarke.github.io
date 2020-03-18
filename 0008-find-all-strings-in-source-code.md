# How to find all strings and comments in source code

## Strings

One of the most useful commands I've developed for reviewing source is a command to extract all strings from a
given directory.  To execute this command, open a terminal and `cd` into the directory containing the source you 
would like to audit.  Next, run:

```
egrep -e "(\"|')(\w|\s|\d)*(\"|')" -r -h -I -o . | sort -u 
```

This will return the ASCII strings for the source in that directory.  A brief description of each argument:

* `-r` - recursively search files in this directory
* `-h` - do not display filename of match
* `-I` - do not search binary files
* `-o` - display only matching text

* `-e '"(\"|')(\w|\s|\d)*(\"|')"'` will match any word, whitespace, or digit character that falls in between single or double quotation marks.
* `sort -u` will sort the results alphanumerically and then only show unique results.

### What to look for

Generally, I use this command to look for credentials hardcoded in source code.  Credentials normally stand out because they 
are "more random" than most strings, while at the same time usually containing only ASCII characters that can also be out of the hex range (which distinguishes them from hashes) and not base64 encoded.  If you see "1337 sp34k" in the results, that can be  a good indicator of a password.  

Other use-cases include looking for HTTP endpoints for server communications/routes and hardcoded cryptographic keys.

## Comments

Comments can contain useful information about code and sometimes even include credentials.  Here is a command to run to see
all the comments in a folder full of source code:

```
egrep -e '^(\/\/|\#\s|\/\*((.|\n|\r)*)\*\/)' -r -I -n .
```

A brief description of arguments:

`-n` - shows the line number of the file where the result was found.

### What to look for

The strings `todo` or `fixme` can be useful for finding bugs that the developer is aware of but has not yet fixed:  

```
egrep -e '^(\/\/|\#\s|\/\*((.|\n|\r)*)\*\/).*(todo|fixme)' -r -I -n -i .
```

A brief description of arguments:

`-i` - search is case-insensitive, so in this case it would match both TODO and todo (and Todo).

[Back](https://nstarke.github.io/)