# E5 A轨道 BuildChecker 人工样例复现说明

实验日期：2026-10-13。执行者：助教本机人工演练。证据来源：用户回传的真实终端输出，尚未由助手独立执行。结论为人工解读原始跟踪记录，不是自动检测器输出。

## 实验输入与环境

- 初始模板提交：06e9a1a。
- Ubuntu工作仓库：/home/hello/devops-course/20261020/A-buildchecker。
- 容器内实验副本：/app/work/e5-a-cqofdC；映射为工作仓库的work/e5-a-cqofdC。
- 镜像：e4-buildchecker:TA1020001。
- 镜像ID：sha256:5af6fc2b8e87e2f461132efb2103ada4ebb089d8b2292e4f5a477953027045af。
- 容器工具：GCC 14.2、GNU Make 4.4.1、strace 6.13、Python 3.13.14。
- 被测数据：课程人工md-rd样例，聚焦main.o、本次构建配置、项目内文件。

Ubuntu宿主机与课程容器的工具版本不同；本次跟踪分析采用上述容器版本。实验副本尚无独立Git提交，因此模板commit不能替代副本文件内容记录。

## 已完成的工程验证

A镜像构建成功；指定镜像的pytest输出3 passed并正常返回0；make scan报告未发现问题；make smoke报告make_exit_code=0、app_output=1、config_h_opened命中、unused_h_opened为空、passed=true。

首次make test输出通过后被Ctrl+C中断，未计为正常通过。随后先复测了Compose默认local镜像，又用明确的e4-buildchecker:TA1020001执行pytest，后者3项测试通过且退出码0。证据中保留这一区别。

## 原始记录采集

在独立样例副本执行make clean，再运行：

```bash
strace -ff -s 4096 -o trace/t -e trace=%file,%process make -p app > make-db.txt 2> build-stderr.log
```

构建退出码为0，运行./app输出1。副本中执行过两轮采集，第一轮PID为15—21，第二轮PID为31—37。make-db.txt被第二轮覆盖；本说明只使用第二轮t.31—t.37，不将两轮混为一次实验。

## 进程归属与目标创建

| 进程 | 角色 | 原始证据 |
|---|---|---|
| 31 | make -p app | t.31第1行execve成功 |
| 32 | cc -c main.c -o main.o编译作业 | t.31第55行clone返回32；t.32第1行execve成功 |
| 33 | cc1编译子进程 | t.32第64行clone返回33；t.33第1行execve成功 |
| 34 | as汇编子进程 | t.32第84行clone返回34；t.34第5行execve成功 |
| 35—37 | cc→collect2→ld链接作业 | t.31第66行clone返回35，链接命令main.o→app，独立于32编译作业 |

与目标依赖有关的关键记录：

- t.33第580行：openat(AT_FDCWD, "config.h", O_RDONLY|O_NOCTTY) = 4。
- t.34第23行：openat(AT_FDCWD, "main.o", O_RDWR|O_CREAT|O_TRUNC, 0666) = 3。
- make-db.txt第1063行：main.o: main.c unused.h。
- 第二轮t.31—t.37未检索到打开unused.h的open/openat记录。

33和34的共同父进程为32。虽然读取头文件与写入目标文件不是同一个PID，它们属于同一编译作业；因此可将config.h关联到该作业产物main.o。不能要求同一个PID既读头文件又写main.o，否则会漏掉GCC这类多进程工具链。

## 本样例结论

| 类型 | 目标 | 依赖 | 判断依据 |
|---|---|---|---|
| MD 缺失依赖 | main.o | config.h | 编译作业成功只读打开该项目头文件，声明前置依赖中没有它 |
| RD 冗余依赖 | main.o | unused.h | 声明中包含它，固定样例本次构建未观察到编译作业打开它；结合课程样例源代码核对 |

RD判断限定课程人工样例和当前构建配置。未出现open调用本身不能证明任意真实项目的文件在所有配置下都不需要，也不能自动据此删除声明。这里不是论文真实项目召回率或误报率评估。

## 追踪中正常出现的失败调用

clone3返回ENOSYS后，程序通过clone成功创建子进程。本次构建退出码0；这种回退不等于编译失败。as在若干PATH位置execve返回ENOENT后，最终在/usr/bin/as执行成功。自动解析时应只把成功的进程创建返回值用于父子关系，不能将失败的路径尝试当成成功运行。

## 与论文方法及后续阶段的关系

E5完成一次clean build原始记录、GNU Make声明数据库及人工依赖推导。论文BuildChecker采用ptrace实现跟踪；本次课程用strace采集系统调用，并人工完成进程作业归属。尚未实现通用的路径归一化、多目标归属、实际图与声明图自动构造和MD/RD自动报告。

E6将上述分析转换为可运行工具，替代人工grep和推导；E7增加正常及故障样例、路径和进程边界测试、重复运行与CI阻断验证。现有3项测试只覆盖E4骨架，不作为核心算法已验证的依据。

方法依据：用户提供的TSE BuildChecker论文PDF第4—6页；课程原设计§4；此前E5课件的A轨道跟踪、声明提取与比较步骤。
