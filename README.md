# linux学习心得、总结

## 软件包安装管理
### 1、源码

   **位置**
  > /usr/src内核源代码
  
   /usr/local用户下载得源代码，如/usr/local/pycharm
      
   **解包**
   
  > tar -zxvf pycharm.tar.gz  -C /usr/local
  
   tar -jxvf pycharm.tar.bz2 -C /usr/local
      
   **安装源码软件**
    
  > ./configure --prefix=/usr/local/pycharm   #软件配置，可以看帮助文件 ./configure --help |more
  
   make  #编译
   
   make install  #安装
      
   **升级**
     
  > diff -Naur /usr/local/pycharm/old.file /下载/pycharm/new.file >diff.patch
  
   patch -pn < diff.patch  
   
   -pn用来同步两个目录，补丁文件记录得目录中取消几个/，n就是几，此例为2,取消了“/”及"/下载“两个文件夹，依赖补丁文件，其中记录了文件所在；
   简单点直接将补丁文件复制到需要打补丁的文件夹中，n取0.
   
   **然后 make、mke install**
      
   **取消升级**
  > patch -R < diff.patch
      
   **源码包卸载**
  > rm -rf /usr/local/pycharm
    
## 2、包管理器 pacman、yay

  **常用**
    -Syyu       强制升级系统，需要root权限
    -S          安装、升级软件包、包组(包含大量软件，可选择序号安装，^表示非，如^2，不安装第2个软件)，需要root权限
    -Rsn        卸载软件并清楚不需要的依赖，需要root权限
    -Sc         清除缓存、保留有效版本，需要root权限
    --needed    避免已有的包重复安装  
    
    -Ss         查询云端软件包简略信息，不需要精确软件信息，包含是否已安装
    -Si         查询包详细信息，不一定是已安装的包，需要准确软件名
    -Qi         查询本地已安装软件详细信息、依赖关系，需要准确软件名，可通过Qe查询，或不接软件名
    -Qs         查询本地已安装软件名字作用、依赖关系，需要准确软件名，可通过Qe查询，或不接软件名 
    
    -Q          查询本地已安装的软件名字
    -Qe         所有明确安装的软件，部需要接软件名
    -Qkk        查询软件文件有无变化，”| grep 有变化"
    -Qk         查询软件文件有无缺失
    
    -F          查询包含某软件包的软件库及安装位置，记忆模糊时可用，如pacman -F pycharm
    -Ql         查询本地包文件列表、安装位置         
    -Fl         查询远程包文件列表、安装位置  
    -Qo         某文件系统由某软件包所拥有
    -Qdt        查询所有不再作为依赖的软件包（孤儿），不需要接软件名
    -Qet        查询所有明确安装且不被需要的软件包，不需要接软件名
    
    pactree     依赖树
     -r         一个安装的软件包被那些包依赖   
    
  **参数**
  
    -S           sync同步
    -R           remove删除
    -Q           query查询
    -U           upgrade升级,可安装本地包
    -F           file查询文件数据库
    -Sw          下载包但不安装
    -Fy          同步文件数据库
    -c           clean
       
   **-Q的参数**
   
    -y           refresh
    -k           check
    -o           owner
    -i           info
    -l           list
    -s           search
    -e           explicit明确的
    -d           dependent
    -t           unrequired
    -c           changlog
   
   > find ~ -name python.py -a -size 10k -o -mtime -7 -not -perm 755 -o -user aria -a -type f
   
   > find / -size 100M -ok rm -rf {} \;         #需要询问才执行
   
   > find / -name *.bac -exec rm -rf {} \;      #直接执行
    
    
  
  
    
