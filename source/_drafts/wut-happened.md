---
title: 发生甚么事了
---

```shell
for i in {01..10}; do
    echo "\begin{problem}" > "prob${i}.tex"
    echo >> "prob${i}.tex"
    echo "\end{problem}" >> "prob${i}.tex"
done
```
