# Comment format

Each finding looks like this, so what you see here matches what a reviewer will write:

```
1. <short title>   issue (blocking) | suggestion | question | nit   Rule: <ID or file#heading>
   File: path/File.java:42
   What: <what is wrong>
   Why it matters: <real consequence>
   How to fix: <concrete change>
```

- Number the findings so you can tick them off.
- Blocking = would stop a reviewer from approving (bug, security, missing requirement, test weakened to pass).
- Non-blocking = a reviewer would suggest it but could merge without it.
- At most five nits.
