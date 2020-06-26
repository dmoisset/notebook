# My reading of PEP-622

After reading the recently proposed [PEP-622](https://www.python.org/dev/peps/pep-0622/) and the discussion that ensued in the [python-dev](https://mail.python.org/archives/list/python-dev@python.org/thread/RFW56R7LTSC3QSNIZPNZ26FZ3ZEUCZ3C/) mailing list I wanted to write some notes. Both to organise my own thoughts but also to share with other people discussing this topic.

# About syntax and clarity

|Pattern               |example       |target-like|expression-like|
|----------------------|--------------|:---------:|:-------------:|
| name_pattern         |`foo`         |✔️|✔️|
| literal_pattern      |`42`          |✖️|✔️|
| constant_pattern(A)  |`Color.BLACK` |✔️|✔️|
| constant_pattern(B)  |`.BLACK`      |✖️|✖️|
| sequence_pattern     |`[x,y]`       |✔️*|✔️*|
| mapping_pattern      |`{"x": x}`    |✖️|✔️*|
| class_pattern        |`Point(x,y)`  |✖️|✔️*|

\* Only if the component sub-patterns are correspondingly expression-like/target-like

- .constant_pattern is weird
- evrything looks like an expression
- half the things do not look like targets
