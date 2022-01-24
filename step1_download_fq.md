> 以[GSE168522](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE168522)为例

# 一、下载fq数据

## 1、获取SRR_Acc_List.txt

- 进入到GSE信息描述主界面，拉到最下面
- 如果有如下图红框标注的文字，说明作者有上传原始数据，点击`SRA Run Selector`

<img src="https://s6.jpg.cm/2022/01/23/Lp6wxf.png" alt="Lp6wxf.png" style="zoom:50%;" />

- 进入SRR文件信息介绍界面，点击`Accession List`，下载所有样本的SRR id文本信息

<img src="https://s6.jpg.cm/2022/01/23/Lp60ti.png" alt="Lp60ti.png" style="zoom:50%;" />

```shell
cat SRR_Acc_List.txt
# SRR13911909
# SRR13911910
# SRR13911911
# SRR13911912
# SRR13911913
# SRR13911914
```

## 2、下载测序数据fastq.gz

```shell
# 以SRR13911909为例：
conda activate download

id=SRR13911909
pare_dir=/home/ssli/scRNA
```



### 2.1 aspera下载

- 使用aspera可以达到百兆的速度，建议首先尝试。但最近试了几次，容易报错，不稳定(报错内容如下)；有时候可以。

  ```shell
  # ascp: failed to authenticate, exiting.
  # Session Stop  (Error: failed to authenticate)
  ```

- https://www.ebi.ac.uk/ena/browser/view/SRR13911909

![LpbNBu.png](https://s6.jpg.cm/2022/01/24/LpbNBu.png)

```shell
# era-fasp@fasp.sra.ebi.ac.uk:/vol1/fastq/SRR139/009/SRR13911909/SRR13911909_1.fastq.gz
# era-fasp@fasp.sra.ebi.ac.uk:/vol1/fastq/SRR139/009/SRR13911909/SRR13911909_2.fastq.gz
# 观察上述链接规律后，可以自动生成下载链接
```



```shell
ascp -QT -l 300m -P33001  \
-i ~/miniconda3/envs/download/etc/asperaweb_id_dsa.openssh   \
era-fasp@fasp.sra.ebi.ac.uk:/vol1/fastq/${id:0:6}/0${id:0-2}/${id}/${id}_1.fastq.gz  ${pare_dir}/rawfq/

ascp -QT -l 300m -P33001  \
-i ~/miniconda3/envs/download/etc/asperaweb_id_dsa.openssh   \
era-fasp@fasp.sra.ebi.ac.uk:/vol1/fastq/${id:0:6}/0${id:0-2}/${id}/${id}_2.fastq.gz  ${pare_dir}/rawfq/
```



> 如上图所示，ebi网站也提供了fastq.gz的ftp链接。但经过尝试后，不建议使用。因为下载不稳定，容易中断(网速极慢)。尽管`wget`支持断点续传，但是会出现部分序列丢失情况，导致后续步骤报错。当aspera方法暂时不能使用时，建议使用下面介绍的`prefetch`方法。
>



### 2.2 preftech下载

- 先下载`.sra`文件，

```shell
# -p 显示进程
# -X 允许下载的文件内存上限
# -O 下载文件的储存路径
prefetch -p -X 35G SRR13911911 -O ${pare_dir}/rawfq/
```

- 再拆分为`fastq.gz`文件

```shell
fasterq-dump --split-files ${pare_dir}/rawfq/$id -p -O --split-files ${pare_dir}/rawfq

# pigz多线程压缩
cd ${pare_dir}/rawfq
ls *fastq | while read id
do
pigz -v $id
done

#注意一下：fasterq-dump拆分出的文件可能不是fq.1，fq.2；而是fq.2，fq.3。在后面重命名时需要注意以下
ls -lh ${pare_dir}/rawfq/*gz
# -rw-r--r-- 1 ssli ssli 8.8G Jan 24 14:14 /home/ssli/project/sxjns/02_AD_Bcell/rawfq/SRR13911911_2.fastq.gz
# -rw-r--r-- 1 ssli ssli  33G Jan 24 14:16 /home/ssli/project/sxjns/02_AD_Bcell/rawfq/SRR13911911_3.fastq.gz

# mv ${pare_dir}/rawfq/${id}_2.fastq.gz ${pare_dir}/rawfq/${id}_S1_L001_R1_001.fastq.gz
# mv ${pare_dir}/rawfq/${id}_3.fastq.gz ${pare_dir}/rawfq/${id}_S1_L001_R2_001.fastq.gz
```



### 2.3 fastq.gz文件重命名

https://support.10xgenomics.com/single-cell-gene-expression/software/pipelines/latest/using/fastq-input

文库共有3种read：

- I1：sample index，用于区分不同（客户）样本的标签
- R1：Barcode + UMI
- R2：单细胞的基因数据（我们想要测到的数据）

在分析数据时，有时是只有R1与R2数据，但对于后续cellranger比对是足够的了，也不需要I3标签数据

<img src="https://s6.jpg.cm/2022/01/24/LpuDIQ.png" alt="LpuDIQ.png" style="zoom: 67%;" />

需要将R1、R2的fastq.gz文件重命名为cellranger可以识别的格式。

```
PROJECT_FOLDER
|-- MySample_S1_L001_I1_001.fastq.gz
|-- MySample_S1_L001_R1_001.fastq.gz
|-- MySample_S1_L001_R2_001.fastq.gz
|-- MySample_S1_L002_I1_001.fastq.gz
|-- MySample_S1_L002_R1_001.fastq.gz
|-- MySample_S1_L002_R2_001.fastq.gz
```

```shell
mv ${pare_dir}/rawfq/${id}_1.fastq.gz ${pare_dir}/rawfq/${id}_S1_L001_R1_001.fastq.gz
mv ${pare_dir}/rawfq/${id}_2.fastq.gz ${pare_dir}/rawfq/${id}_S1_L001_R2_001.fastq.gz

ls ${pare_dir}/rawfq/
# SRR13911909_S1_L001_R1_001.fastq.gz  SRR13911909_S1_L001_R2_001.fastq.gz

zcat SRR13911909_S1_L001_R1_001.fastq.gz | head -4
# @SRR13911909.1 E00512:203:HFJYTCCXY:6:1101:5071:1907/2
# CNAGATCCACGGCCATCATCAAAAGT
# +
# <#AFFJFJJJJJJJJJJFJFJJJJJJ

zcat SRR13911909_S1_L001_R2_001.fastq.gz | head -4
# @SRR13911909.1 E00512:203:HFJYTCCXY:6:1101:5071:1907/3
# NCACGTGGTGTGTTTCGTGGGAACAACTGGGCCTGGGATGGGGCTTCACTGCTGTGACTTCCTCCTGCCAGGGGATTTGGGGCTTTCTTGAAAGACAGTCCAAGCCCTGGATAATGCTTTACTTTCTGTGTTGAAGCACTGTTGGTTGTTT
# +
# #AA7FFJFJJJJJJJAFF-FJFJAJJFAJFJJ-FJJJJFFFJJ<FFFJJJJJJJJJJFJJAFJJJJJFJFJJJJ-77FJJFJJJJJFJJ77<FJJJ7FJAJFFF7F7-AFF7AFFJFJJJJFJJF<JFFFJJ<FAJ<F-<AJAF-AFFJJJ
```



# 二、cellranger比对

## 1、下载软件与参考基因组

https://support.10xgenomics.com/single-cell-gene-expression/software/downloads/latest?

- 软件（2022.01.23 最新版本为6.1.2）

  ```shell
  wget -c -O cellranger-6.1.2.tar.gz "https://cf.10xgenomics.com/releases/cell-exp/cellranger-6.1.2.tar.gz?Expires=1642953993&Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9jZi4xMHhnZW5vbWljcy5jb20vcmVsZWFzZXMvY2VsbC1leHAvY2VsbHJhbmdlci02LjEuMi50YXIuZ3oiLCJDb25kaXRpb24iOnsiRGF0ZUxlc3NUaGFuIjp7IkFXUzpFcG9jaFRpbWUiOjE2NDI5NTM5OTN9fX1dfQ__&Signature=WFkGa82mNDuSm9lWkgGiLuVGvM1jIwO~LElA5RJB7ZuBUf74NagCTtDPAv3GVI8XMwCMyyWYu46tX6Fy4u2NrbjwWgU133JDoJsKQDgkGyFXPwNQjQ544yc3-3oZGWnSGplIoyWCgQ7wtmB0MTRBi1a4LGBT4K3gyOAsA8oaWALGqqabXfMYB7GxANRodkMWKkgTJ7YR4HIciQQZGDpY0h05J0MpU2m5YgHsBaQXnmBBxzJYucIdI5-NcbetYuou9Bh3IY48qTOce9xJ9Kk5sOJSN19qpIXHN47VU62v-mgLjwKMZ6BSt~5tZZLgNO1INUKH27oy4c2VttR8QaQ3tg__&Key-Pair-Id=APKAI7S6A5RYOXBWRPDA"
  
  #解压即用
  tar -zxvf cellranger-6.1.2.tar.gz
  
  /home/ssli/software/cellranger-6.1.2/bin/cellranger --version
  # cellranger cellranger-6.1.2
  ```

- 参考基因组，有人/鼠可选

  ```shell
  wget -c https://cf.10xgenomics.com/supp/cell-exp/refdata-gex-GRCh38-2020-A.tar.gz
  
  tree -L 1 /home/ssli/software/cellranger-6.1.2/ref/refdata-gex-GRCh38-2020-A
  # /home/ssli/software/cellranger-6.1.2/ref/refdata-gex-GRCh38-2020-A
  # ├── fasta
  # ├── genes
  # ├── pickle
  # ├── reference.json
  # └── star
  ```

  

## 2、cellranger比对

```shell
bin=/home/ssli/software/cellranger-6.1.2/bin/cellranger 
ref=/home/ssli/software/cellranger-6.1.2/ref/refdata-gex-GRCh38-2020-A 

cd ${pare_dir}

$bin count \
--fastqs=${pare_dir}/rawfq \
--sample=${id} \
--transcriptome=$ref \
--id=${id} \
--localcores=10 \
--no-bam --nosecondary 

# --fastqs 交代测速数据路径
# --sample 交代待比对样本名的前缀（因为该文件夹内可能有许多样本的测序数据）
# --transcriptome 交代参考基因组文件夹
# --id 交代储存结果的文件夹，如果没有会自动创建
# --localcores 多线程
# --no-bam 不生成bam文件
# --nosecondary 不进行后续分析
```



# 三、流程脚本

由于cellrange比较耗内存，所以即用即删，仅保留所需要的表达数据文件。

对于每一个SRR文件：

（1）下载fastq文件；（2）cellranger比对；（3）保留表达数据，删除其余数据以及fastq.gz数据

```shell
bin=/home/ssli/software/cellranger-6.1.2/bin/cellranger 
ref=/home/ssli/software/cellranger-6.1.2/ref/refdata-gex-GRCh38-2020-A 
pare_dir=/home/ssli/scRNA

conda activate download
cat SRR_Acc_List.txt
# SRR13911909
# SRR13911910
# SRR13911911
# SRR13911912
# SRR13911913
# SRR13911914

id=SRR13911912
```



## 1、aspera流

如果aspera可以下载，那么优先使用这种方法

```shell
cat > aspera_pipeline.sh

#!/bin/bash
bin=$1
ref=$2
pare_dir=$3
id=$4

#(1)下载fastq文件
## fq1
ascp -QT -l 300m -P33001  \
-i ~/miniconda3/envs/download/etc/asperaweb_id_dsa.openssh   \
era-fasp@fasp.sra.ebi.ac.uk:/vol1/fastq/${id:0:6}/0${id:0-2}/${id}/${id}_1.fastq.gz  ${pare_dir}/rawfq/
## fq2
ascp -QT -l 300m -P33001  \
-i ~/miniconda3/envs/download/etc/asperaweb_id_dsa.openssh   \
era-fasp@fasp.sra.ebi.ac.uk:/vol1/fastq/${id:0:6}/0${id:0-2}/${id}/${id}_2.fastq.gz  ${pare_dir}/rawfq/
## rename
mv ${pare_dir}/rawfq/${id}_1.fastq.gz ${pare_dir}/rawfq/${id}_S1_L001_R1_001.fastq.gz
mv ${pare_dir}/rawfq/${id}_2.fastq.gz ${pare_dir}/rawfq/${id}_S1_L001_R2_001.fastq.gz

#(2)cellranger比对
cd ${pare_dir}
$bin count \
--fastqs=${pare_dir}/rawfq \
--sample=${id} \
--transcriptome=$ref \
--id=${id} \
--localcores=10 \
--no-bam --nosecondary 

#(3)保留表达数据，删除其余数据以及fastq.gz数据
cd ./${id}/
mv ./outs/filtered_feature_bc_matrix ./
## rm cellranger中间数据
rm -rf !(filtered_feature_bc_matrix)
## rm fatsq.gz文件
cd ${pare_dir}
rm -rf ${pare_dir}/rawfq/${id}*
```

- 执行脚本

```shell
chmod u+x aspera_pipeline.sh

bin=/home/ssli/software/cellranger-6.1.2/bin/cellranger 
ref=/home/ssli/software/cellranger-6.1.2/ref/refdata-gex-GRCh38-2020-A 
pare_dir=/home/ssli/scRNA
id=SRR13911912

./aspera_pipeline.sh $bin $ref $pare_dir $id
```



## 2、prefetch流

与上面的区别，就是第一步下载fastq数据方式不同

```shell
cat > prefetch_pipeline.sh

#!/bin/bash
bin=$1
ref=$2
pare_dir=$3
id=$4
#(1)下载fastq文件
## sra
prefetch -p -X 35G ${id} -O ${pare_dir}/rawfq/
## fastq
fasterq-dump --split-files ${pare_dir}/rawfq/${id} -p -O ${pare_dir}/rawfq
## fastq.gz
cd ${pare_dir}/rawfq
ls *fastq | while read id
do
pigz -v $id
done
## rename(2→1, 3→2)
mv ${pare_dir}/rawfq/${id}_2.fastq.gz ${pare_dir}/rawfq/${id}_S1_L001_R1_001.fastq.gz
mv ${pare_dir}/rawfq/${id}_3.fastq.gz ${pare_dir}/rawfq/${id}_S1_L001_R2_001.fastq.gz

#(2)cellranger比对
cd ${pare_dir}
$bin count \
--fastqs=${pare_dir}/rawfq \
--sample=${id} \
--transcriptome=$ref \
--id=${id} \
--localcores=10 \
--no-bam --nosecondary 

#(3)保留表达数据，删除其余数据以及fastq.gz数据
cd ./${id}/
mv ./outs/filtered_feature_bc_matrix ./
## rm cellranger中间数据
rm -rf !(filtered_feature_bc_matrix)
## rm fatsq.gz文件
cd ${pare_dir}
rm -rf ${pare_dir}/rawfq/${id}*
```

- 执行脚本

```shell
chmod u+x prefetch_pipeline.sh

bin=/home/ssli/software/cellranger-6.1.2/bin/cellranger 
ref=/home/ssli/software/cellranger-6.1.2/ref/refdata-gex-GRCh38-2020-A 
pare_dir=/home/ssli/scRNA
id=SRR13911912

./prefetch_pipeline.sh $bin $ref $pare_dir $id
```

