# use issue
第一次使用cgroup的时候会出现如下问题:`No space left on device`,过程如下:
```
mkdir cgroups/A
echo 500 > cgroups/A/cpu.shares
echo 3167 > cgroups/A/tasks
bash: echo: write error: No space left on device
```
解决方法:
```
echo 0 >cgroups/A/cpuset.mems
echo 0 >cgroups/A/cpuset.cpus
```
看起来cgroup在mount的时候默认应该选中当前的所有CPU和memory节点，不过它好像没有做  
