# Search for Regex with Regex

This is a command I developed to search for regular expressions using egrep:

```bash
egrep -r -e "[\^]?(\.\*)|(\[\w+*\-\w+*\])|(\{\d+[\,\d+]?\})[\$]?" .
```

[Back](https://nstarke.github.io/)