# 概述
**eggNOG-mapper** 是一款能够快速对大量蛋白质序列进行功能注释的软件。它使用eggNOG数据库提供的直系同源（orthology）与系统发育学（phylogenetics）数据库，将直系同源蛋白的注释映射到陌生蛋白中。

该软件所构建的的直系同源功能映射方法在精确度上要明显优于其它传统的同源搜索方法（比如，BLAST搜索和Interproscan搜索），因为其排出了很多相似的旁系同源（paralogy）蛋白质（该类蛋白质的功能很可能有明显的分化）。

软件的作者提供了详细的eggNOG-mapper与BLAST和Interproscan的比较结果。[点击这里](https://github.com/jhcepas/emapper-benchmark/blob/master/benchmark_analysis.ipynb).

# 安装
## 软件需求
+ Linux 系统
+ Python 2.7，目前eggNOG-mapper不支持Python 3版本
+ DIAMOND 软件（一般无需安装，除非是有最新版本，因为eggNOG-mapper已打包了该DIAMOND的可执行文件）
+ BioPython包

## 存储空间需求
+ 至少40GB的空间存放eggNOG注释数据库（eggnog.db）
+ 至少10GB的空间存放序列数据库（eggnog_proteins.dmnd）

## 下载
+ 可以直接从Github网站上下载最新版本的eggNOG-mapper：
(https://github.com/jhcepas/eggnog-mapper/releases)。解压缩后即可运行，无需编译和安装。
+ 或者从官方仓库（主分支）中克隆下来一份到本地：
`git clone https://github.com/jhcepas/eggnog-mapper.git`

## 获取数据库
+ 官方提供了自动批量下载两个数据库的脚本，运行如下命令：

  `download_eggnog_data.py `

  它会将数据库下载下来并且解压缩到eggNOG-mapper软件目录中的**data**目录中。
+ 不过最好使用专业的下载工具来下载数据库，因为服务器下载很慢。目前已下载解压最新的数据库并存放在：

  `/home/lxc/bin/eggNOG-mapper/data`
 
## 基础使用
+ 只需要提供需要注释的所有蛋白质序列的文件（FASTA格式），即可运行`emapper.py`：

  `python emapper.py -i test.fasta --output test -m diamond --no_file_comments`

+ 建议将eggNOG-mapper解压后的主目录加入到环境变量中，方便在任意路径下运行调用。
+ 必须选择的参数有：

  `-i 输入蛋白质文件的相对路径或绝对路径`；
  
  `-m 必须选择diamond`；
  
  `--output 输出文件的标题（title）`；
  
+ 建议选择的参数有：
  
  `--cpu 20+ 最好20个核以上，对大文件效果显著`
  `--no_file_comments 输出文本文件的开头没有没用的comment，利于后续程序的读写`
  `--overide 此命令会忽视已存在的注释结果文件直接覆盖写入，加上程序运行更加安全`
  
+ 输出文件：
 
   1、`*.emapper.annotations` 最终获得的所有蛋白的注释结果
   结果主要是这个文件，每行是一个蛋白质注释结果，总共有22列注释信息，其中COG/KOG号为第19列（倒数第4列），结果类似
   
   `35M44@314146,3AIXI@33154,3BZ60@33208,3D3B7@33213,3J9T0@40674,488P6@7711,498IJ@7742,4M93N@9443,4MVXJ@9604,COG0526@1,KOG0907@2759`
   
   这里只需提取最后一个@号前面的那个KOG号即可。
   
   对应的COG/KOG类别为第21列（倒数第2列），可以直接用于统计。
   
   2、`*.emapper.seed_orthologs` 所有蛋白质的直系同源比对结果


## 大规模蛋白质组注释建议
如果需要注释的蛋白质数量特别多，比如超过100万个蛋白质，可考虑以下的建议：

eggNOG-mapper的整个运行过程分成两大部分，分别是：1）直系同源搜索，2）同源注释。其中第1步是计算密集型，需要更多CPU的参与；第2步则是读写密集型，对硬盘的读写要求更高。

因此可以分别进行优化：

#### 步骤1. 同源搜索
1. 将单个蛋白质组序列文件等比拆分成若干个分文件，可以自己写脚本，也可以运行如下Linux命令：

    `split -l 2000000 -a 3 -d input_file.faa input_file.chunk_`
2. 使用DIAMOND模式，每个分文件可以单独运行在一个计算任务中。而且，需要设置eggNOG-mapper不要运行注释步骤。此过程可以写程序批量运行，也可以使用如下的Linux命令：

    `# generate all the commands that should be distributed in the cluster`
    
    `for f in *.chunk_*; do`
    
    `echo ./emapper.py -m diamond --no_annot --no_file_comments --cpu 16 -i $f -o $f;`
    
    `done` 
    
#### 步骤2. 功能注释
此步骤注释需要密集地查询`data/eggnog.db`，因此IO是主要的限速步骤。这个数据库是一个sqlite3类型数据库，因此建议将其存放在读写更快的硬件中，如固态硬盘，效果会好很多。
承接上一步：
3. 串联所有直系同源结果：

    `cat *.chunk_*.emapper.seed_orthologs > input_file.emapper.seed_orthologs`

4. 进行同源功能注释：

    `emapper.py --annotate_hits_table input.emapper.seed_orthologs --no_file_comments -o output_file --cpu 20`


