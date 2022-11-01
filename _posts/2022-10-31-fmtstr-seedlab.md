---
layout:       post
title:        "SEED Labs 2.0: Format String Attack Lab Writeup"
category:     "Computing Systems"
tags:         system-security docker reverse-shell
permalink:    /fmtstr-seedlab/
---

For general overview and the setup package for this lab, please go to [SEED Labs official website](https://seedsecuritylabs.org/Labs_20.04/Files/Format_String/). The lab assignment was conducted using SEED virtual machine configured on a AWS EC2 instance along with containers. The set-up instructions for Docker Compose can be found [here](https://github.com/seed-labs/seed-labs/blob/master/manuals/docker/SEEDManual-Container.md)

The **Format Function**{: style="color: red"} is an ANSI C conversion function, like <code>printf</code>, <code>fprintf</code>, which converts a primitive variable of the programming language into a human-readable string representation. The **Format String**{: style="color: red"} is the argument of the Format Function and is an ASCII Z string which contains text and format parameters, like:
```c
printf("The magic number is: %d\n", 1911);
```
The **Format String Parameter**{: style="color: red"}, like <code>%x</code> or <code>%s</code>, defines the type of conversion of the format function[^1].

<!-- excerpt-end -->

|---
| | **Buffer Overflow** | **Format String**
|:-|:-|:-
| **Public since** | mid 1980's | June 1999
| **Danger realized** | 1990's | June 2000
| **Number of exploits** | a few thousands | a few dozen
| **Considered as** | security threat | programming bug
| **Techniques** | evolved and advanced | basic techniques
| **Visibility** | sometimes very difficult to spot | easy to find

*Table 1: Buffer Overflow vs. Format String Vulnerabilities, from [Team TESO's 2001 paper](https://cs155.stanford.edu/papers/formatstring-1.2.pdf)*[^2]
{:.table-caption}

## References

[^1]: See [owasp.org](https://owasp.org/www-community/attacks/Format_string_attack).

[^2]: scut/team teso, "Exploiting Format String Vulnerabilities," September 1, 2001.