---
title: 使用Modelsim中的CommandTools 进行仿真
author: dashjay
date: '2019-07-27'
slug: modelsim-commandtools
categories: []
tags:
  - verilog
  - HDL
type: ''
subtitle: ''
image: ''
---

> 之前一直使用的是iverilog进行的仿真，生成vvp可执行文件之后运行得到结果
方法如下直接给Makefile

```
all: cmp vvp lxt

cmp:
	iverilog *.v -o tb_top.vvp
vvp:
	vvp tb_top.vvp

lxt:
	cat top.lxt

clean:
	@rm -rf tb_top.vvp tb_top.lxt
```
有了以上`Makefile`，将`*.v`文件和`Makefile`放在一起，make即可完成相应操作

但是用这种方式，生成的波形只能使用GTKwave来查看，并且当有一些复杂的情况出现时候，iverilog就无法处理了，所以我们使用Intel提供的免费的Modelsim来完成仿真，仿真资料比较少，全是图形化界面的介绍，道路坎坷。

### 第一步：下载全套工具
intel官网下载的lite版本，最小套装
`Quartus-lite-18.1.0.625-linux.tar`

### 第二步：安装
解压之后有一键安装脚本，运行的时候会提示缺库，安装对应的库文件就可以使用了。

### 第三步：仿真出波形

#### 1. vlog工具
路径位于`intelFPGA_lite/18.1/modelsim_ase/bin/vlog`
执行`vlog *.v`即可对当前目录所有`.v`文件进行编译操作

我给了一个模板`.v`文件

```
module top_module(
	input [3:0] a,
	input [3:0] b,
	output reg[7:0] p
);

	reg [7:0] pv;
	reg [7:0] ap;
	integer i;

	always@(*)
	begin
		pv = 8'b00000000;
		ap = {4'b0000,a};
		for(i = 0; i<=3; i=i+1)
			begin
				if(b[i]==1)
					pv = pv + ap;
					ap = {ap[6:0],1'b0};
			end
		p = pv;
	end
endmodule
```
运行`vlog *.v`命令即可在当前目录下生成一个work文件夹，里面有一些编译的成果，不过不用太在意文件的具体内容。

#### 2.使用vsim
命令是 `vsim -c -do test.do top_module`
这个test.do文件里面的内容有哪些呢？

```
> cat test.do
add list -hexadecimal /top_module/a
add list -hexadecimal /top_module/b
add list -hexadecimal /top_module/p

do sim.do

write list test.list

quit -f
```
不用太在意具体的内容，知道`-hexadecimal`是十六进制，去掉自动变成十进制。
下面有一个 `sim.do`内容是

```
add wave a
add wave b
add wave p
force a 16#0x8
force b 16#0xa
run 1000
force b 16#0x2
run 1000
force b 16#0x3
run 1000
force b 16#0x4
run 1000
force b 16#0x5
```

相当于，将模块中的`a,b,p`接口抽离出来，接在sim文件当中，强行给了`a,b`输入，给模块提供了测试激励。

我写了一个Makefile 文件如下

```
all: cmp sim cat

cmp:
	vlog *.v
sim:
	vsim -c -do test.do top_module
cat:
	cat test.list
```

那么我把我的文件目录压缩上传提供下载试试


<img src="/post/2019-07-27-modelsim-commandtools.en_files/屏幕快照 2019-07-27 下午2.17.33.jpg" alt="" width="50%"/>

[文件下载](/post/2019-07-27-modelsim-commandtools.en_files/finaltest.zip)

```
.
├── Makefile
├── sim.do
├── test.do
└── top_module.v
```

谢谢