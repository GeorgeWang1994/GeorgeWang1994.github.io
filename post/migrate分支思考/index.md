
<meta name="referrer" content="no-referrer" />

# migrate分支思考

## 1. 问题描述



最近在工作中同事遇到了一些问题，问题如下：



**在master分支中修改了model，并且已经生成了migrations，与此同时，在master分支也更改了model，也生成了migrations，两个分支改动model的地方都是不一样的，后面想将分支系统合并到了master分支，出现了冲突后的解决方案应该怎么办？**



## 2. 解决方案

### 2.1 手动修改



我首先想到的是，将分支系统中的migration的option操作手动都合并到master分支对应的版本的option中，然后再在master分支进行一次migrate，自己手动实验了下，是可行的。

需要改动的地方包括`dependencies`和`operations`，总之就是按照master的migrate的逻辑在更改。



或者直接对更改migration的文件名字为`最新`，并且更改`dependencies`，同样也可以操作。



### 2.2 merge



直接执行`makemigrations`，它会提示出现了冲突，需要进行`migration merge`操作，因此按照它提供的步骤做即可，到时就会出现一个option为空，并且dependencies为两个冲突文件的一个新的文件，如下：



![image](https://wx2.sinaimg.cn/large/007FyU7Tly1g3dn8jmsmbj31dy0hijvm.jpg)



> 如果你觉得这些文件比较冗余，可以通过`squashmigrate`命令将截止到你设置的migration文件合并成一个新的文件，里面包含了历来的更改操作。



以上，进行测试的代码在[Github](https://github.com/GeorgeWang1994/test_migration)能够找到。