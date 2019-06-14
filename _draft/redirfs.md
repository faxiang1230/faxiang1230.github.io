# redirfs
## 背景
它只是提供一个framework,而不管用来做什么。而它目前最复杂的filter是avflt,用来做杀毒防护。
目前不被mainstream的开发人员认可，最开始由目前在github上维护。
comodo antivirus
### 实现原理
#### redirfs object申请和释放
#### filter
## 缺陷
目前redirfs仍然是单独维护的,开发人员包括:,
## 参考

现有的对象hook

新建的对象hook
dentry:
lookup
不管什么操作，都会先查找。无论是否有对应的文件对象，都会创建一个dentry对象。如果没有对应的文件则称为negative dentry.
dentry的作用就是为了快速查找，当你查找不存在的文件时也可以通过缓存dentry来加速。而lookup是dir类型的inode的方法

file新建的hook过程
1.找到路径对应的dentry
2.找到dentry对应的inode
3.拷贝inode->i_fop到file的f_op,之后使用file_operations->open
f->f_op = fops_get(inode->i_fop);
所以hook时机为file_operations->open时,所以hook掉inode的inode_operations,所以问题转化为了在inode创建的时候hook它的i_fop

inode:
不管是file还是dentry，在新创建的时候都聚焦到了inode上。而设置inode的operations还是在新增dentry的时候，这样就好像出现了先有鸡还是先有蛋的问题。肯定先有鸡，这只鸡是我们在初始化的时候手动设置。

对象销毁的时机:
dentry:d_release
file:file_operations->release
inode:super_operations中是各种操作inode创建和销毁的方法，这里我们借助于dentry销毁过程中的回调d_iput

dentry和inode的挂接:
新创建inode:create,mkdir,link,symlink

hook的ops:
1.释放的时候将redirfs关联的对象释放掉
2.埋点给filter使用
