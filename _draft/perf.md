# perf使用

ref:https://github.com/brendangregg/FlameGraph

- 问题1:perf找不到module的符号
```
IN_TREE_DIR=/lib/modules/`uname -r`/kernel/modulename
mkdir -p $IN_TREE_DIR
cp modulename.ko $IN_TREE_DIR
depmod -a
```
