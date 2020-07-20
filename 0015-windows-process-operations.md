# Windows Process Token Bitwise AND To get real value

Published: December 31, 2017

The value stored in the `_EPROCESS` object must be AND'd against a certain value to produce the actual token.
For more information, see [http://mcdermottcybersecurity.com/articles/x64-kernel-privilege-escalation](http://mcdermottcybersecurity.com/articles/x64-kernel-privilege-escalation)

## Windows 32-bit:
```
kd> ?[TOKEN] & 0xFFFFFFF8
```

## Windows 64-bit:
```
kd> ?[TOKEN] & 0xFFFFFFFFFFFFFFF0
```

[Back](https://nstarke.github.io/)
