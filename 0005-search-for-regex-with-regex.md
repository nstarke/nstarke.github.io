# Search for Regex with Regex

Published: February 23, 2016

This is a command I developed to search for regular expressions using egrep:

```bash
egrep -r -e "[\^]?(\.\*)|(\[\w+*\-\w+*\])|(\{\d+[\,\d+]?\})[\$]?" .
```

[Back](https://nstarke.github.io/)