# 序 & 现状
本文总结了在RISC-V上编译sbcl所做过的尝试、实践以及对于某些问题的解决方案

目前Fedora RISC-V上能完美编译（即能交叉编译+自举）的sbcl版本为2.2.3，自从2.2.9版本开始如果不对源码修改则连交叉编译都无法通过

# 名词解释
- SBCL：一个Common Lisp编译器，项目地址位于 https://www.sbcl.org/
- 自举(bootstrap)：在编译器领域，指通过某些特殊方式先构建最基础的编译器，再用该编译器编译代码获得完整功能的软件
- contrib：在SBCL中是扩展库

# 探索过程
## 准备工作
根据官方交叉编译文档，得知sbcl编译需要旧版sbcl自举，对于尚无对应sbcl二进制的系统和平台，需要借助其他已有现成sbcl的系统作交叉编译或使用其他Common Lisp编译器，随后再在target架构上编译contrib

因为RISC-V平台上尚无sbcl可用，因而有两条路径：
1. 调用clisp这种第三方的lisp来编译sbcl
2. 使用其他系统/平台的sbcl来交叉编译RISC-V
据调查，如果使用clisp等第三方编译器，需要特定版本号对应上才能编译成功，而且如果在qemu RISC-V上原生编译sbcl，会花费大量时间（大约3-4小时），因此将重点放在了第二种方案

## 步骤和坑
1. 配置一台riscv虚拟机，假设虚拟机的ip地址是192.168.122.153，并为其配置sshd服务。如果是root用户登录，需要在/etc/ssh/sshd_config中加入：`PermitRootLogin prohibit-password`
2. 假定sbcl源码在虚拟机中的路径是/root/sbcl，在虚拟机中克隆仓库`git clone https://git.code.sf.net/p/sbcl/sbcl`
3. 随后在主机端同样克隆仓库git clone https://git.code.sf.net/p/sbcl/sbcl。
4. 在主机端生成SSH Key，并将key加入虚拟机的authorized_keys中
5. 在主机的sbcl源码目录中运行命令：`sh cross-make.sh sync root@192.168.122.153 /root/sbcl SBCL_ARCH=riscv64 CFLAGS="-fsigned-char"`，`sync`表示虚拟机和主机的sbcl代码保持一致（通过指定虚拟机sbcl源码的git HEAD），最后的是指定在编译时的环境变量
6. 然后就遇到报错
  ```
  + scp root@192.168.122.153:/root/sbcl/remote-version.lisp-expr root@192.168.122.153:/root/sbcl/local-target-features.lisp-expr root@192.168.122.153:/root/sbcl/output/build-id.inc .
  + mv build-id.inc output
  + sh make-host-1.sh
  make-host-1.sh: line 24: output/build-config: Not a directory
  //entering make-host-1.sh
  ```
  通读源码后发现，build-config是包含环境变量的文件，根据再三搜索确定`build-config`应该从虚拟机中生成的，所以在ssh那行命令理应将该文件复制到本地，可是缺缺失了这一段

  而根据上面的那行`mv build-id.inc output`，可以知道作者的意图是将build-id.inc移动到output目录下，但有没有可能刚刚clone下来的源码里并无output目录？这种情况下，build-id.inc就会被重命名成output文件，随后引发又一系列奇怪问题，这个脚本真的是年久失修

  因此这里引入[该patch文件](https://github.com/fedora-riscv/sbcl-build-docs/blob/main/sbcl-cross-make.patch)用于解决这个问题，**同时请务必`mkdir output`确保output目录存在**

7. 修正错误之后重新执行`sh cross-make.sh sync root@192.168.122.153 /root/sbcl SBCL_ARCH=riscv64 CFLAGS="-fsigned-char"`，待主机端编译完毕后，此时虚拟机的sbcl目录上就有个一阶段的sbcl编译器，到虚拟机下的sbcl目录，执行`sh make-target-contrib.sh`
8. 不出意料的话，你将很快看到该脚本喷出了一大堆二进制乱码到你的终端上
9. 在经过大量测试和排障，确定了第一阶段sbcl编译器包含了`contrib/make-contrib.lisp`来构建contrib库，再经过一顿排查，最后确定是`run-program`函数本来是用于将多个文件的内容用cat命令串起来随后输出，但之后发现该函数的output参数失效，即便指定文件也将输出到stdout，进而导致终端被污染
10. 因为`run-program`的output参数失效，利用了些小技巧，创建了[这个patch](https://github.com/fedora-riscv/sbcl-build-docs/blob/main/sbcl-make-contrib.patch)用于绕过这个问题（但`run-program`刚刚提到的问题本身并未被修复！）
11. 应用了这个patch之后发现还是报错，原因竟是output文件夹下没有sbcl-home目录
12. 务必在虚拟机里cd进入sbcl源文件夹后执行`mkdir -p obj/sbcl-home`
13. 在每次编译前务必`./clean.sh`，随后再执行命令，再上述修补操作之后可以编译出初版sbcl二进制和contrib库
14. 编译完成后，将sbcl源文件夹复制一份，如/root/sbcl-new
15. 在新的sbcl文件夹里，执行sh clean.sh，并执行mkdir obj/sbcl-home，随后执行SBCL_ARCH=riscv64 CFLAGS="-fsigned-char" sh make.sh --xc-host='/root/sbcl/run-sbcl.sh' --arch="riscv64"，这里就是调用刚刚编译的初版sbcl编译器自举
16. 然后就能看到一个奇奇怪怪的错误
```
debugger invoked on a TYPE-ERROR @566B4C9C in thread 
#<THREAD tid=30094 "main thread" RUNNING{5439A2C3}>:
  The value
    #S(SB-C:LOCATION-INFO
      :KIND:NON-LOCAL-ENTRY :CONTEXT NIL
      :LABEL#<SB-ASSEM:LABEL 1>:VOP#<SB-C:V0P
      :INFO SB-C:NOTE-ENVIRONMENT-START :ARGS NIL :RESULTS NIL
      :CODEGEN-INFO (#)>)
      
  is not of type 
    SB-C:L0CATION-INFO
Type HELP for debugger help,or (SB-EXT:EXIT) to exit from SBCL.
```
该问题到目前为止暂时无解
