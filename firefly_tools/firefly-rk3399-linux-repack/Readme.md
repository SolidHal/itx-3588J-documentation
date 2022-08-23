1. 解包
    把官方发布的固件拷贝到当前目录，重命名为update.img , 执行unpack.sh
    解包完成后，生成的文件在output目录下.

2. 合包
    保持当前目录结构，文件名等不变，用客户自己的文件替换output/下同名的文件
    执行pack.sh, 执行完后，生成new_update.img，即为打包好的固件
	rootfs文件名必须为rootfs.img
	parameter.txt文件名必须为parameter.txt

注意：
	合包过程中，如果rootfs分区不是最后一个分区，那么程序会根据rootfs文件的大小，
	自动修改parameter.txt中rootfs分区的大小。
	如果用户自己有改动parameter.txt，请留意整个合包的流程。

