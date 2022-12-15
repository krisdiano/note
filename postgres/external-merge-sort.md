# external merge sort

由于数据库会做多表 join 然后可能还要求输出有序结果，因此在这种数据量比较大的情况下，直接在内存中可能无法完成排序，因此需要借助外部排序，排序的算法基于归并排序，本节以二路归并作演示。

算法分别两个部分：

1. sorting: sort blocks of data that fit in main-memory and then write back the sorted blocks to the file on disk.

2. merging: Combing sorted sub-files into a single larger file.

比如不考虑排序算法的程序自身占用的内存可以忽略不急，有 3 个 block 的内存可以使用，要求对 7 个 block 进行排序。

1. 将需要将可使用的内存均分为 3 份，也就是 1 个 block 。分成 3 份是因为 2 个作为输入，1 个作为输出。

2. 根据需要排序的数据量 7 个block 根据可使用的内存均分后的 1 个block 划分为7份，每份 1 个block。

3. 第一轮，对每个 block 单独排序后写回磁盘

4. 第二轮，两两结合，每份都读入内存 1 个 block ，每次 merge 1 block 就写回磁盘，直到两份 merge 完成。

5. 重复步骤4直到只有排序完成。

如图：

![](https://github.com/krisdiano/mdimages/raw/master/cmu445/2022-12-15-20-37-55-image.png)

注意，对于 PASS1 需要将 2 个大小为 2 block 的单位合并成 1 个 4 block 的单位。虽然已经超过了内存的限制，但是这 2 个每次都只读取 1 block，由于只有1 block 做输出，因此这2 个 1 block 数据的 merge 需要写两次盘。

PASS0 的数据都是可以直接排序的，因此每个 block 排序后直接回写磁盘。分析之后的 PASS ，由于输入输出的buffer是固定的，因此对 N 个做 merge 就需要分别读取和回写 N 次。因此`Total I/O cost=2N.(# of passes)` 。


