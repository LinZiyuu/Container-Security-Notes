
# mount(8) command:

Build mount command as follows:
mount [$mountFlags] [-t $fstype] [-o $options] [$source] $target


参数:
-a, --all(挂载fstab中提到的所有fstype)
-B, --bind(bind mount)
**-c, --no-canonicalize**(不规范化路径,默认情况下会对所有路径规范化,一般在确定路径为absolute path时使用)
-F, --fork(并行mount?)
-f, --fake(不太懂没用到)
**-o, --options opts**(用于指定具体的mount选项 opts是一个逗号分隔的列表) 
-R, --rbind(bind remount)
-r, --read-only(也可用-o ro)
--target directory
--source device
-l 列出清单(e.g.: mount [-l] [-t type] 列出所有mounted的filesystem)
**-t fstype**(用于指定文件系统类型)

options:
remount
ro(read-only)
rw(read-write)