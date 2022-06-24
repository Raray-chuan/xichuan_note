## 1. 字节码指令集
### 1.1 概述 
Java虚拟机的指令由一个字节长度的、代表着某种特定操作含义的数字（称为操作码，Opcode） 以及跟随其后的零至多个代表此操作所需参数（称为操作数，Operands）而构成。
比如：
```
字节码 		助记符 			指令含义
0x00 		nop 			什么都不做
0x01 		aconst_null 	将 null 推送至栈顶
0x02 		iconst_m1		将 int 型 -1 推送至栈顶
0x03 		iconst_0 		将 int 型 0 推送至栈顶
0x04 		iconst_1 		将 int 型 1 推送至栈顶
0x05 		iconst_2 		将 int 型 2 推送至栈顶
0x06 		iconst_3 		将 int 型 3 推送至栈顶
0x07 		iconst_4 		将 int 型 4 推送至栈顶
0x08 		iconst_5 		将 int 型 5 推送至栈顶
0x09 		lconst_0 		将 long 型 0 推送至栈顶
0x0a 		lconst_1 		将 long 型 1 推送至栈顶
0x0b 		fconst_0 		将 float 型 0 推送至栈顶
0x0c 		fconst_1 		将 float 型 1 推送至栈顶
0x0d 		fconst_2 		将 float 型 2 推送至栈顶
0x0e 		dconst_0 		将 double 型 0 推送至栈顶
0x0f 		dconst_1 		将 double 型 1 推送至栈顶
0x10 		bipush			将单字节的常量值（Byte.MIN_VALUE ～ Byte.MAX_VALUE，即-128～127）推送至栈顶
0x11 		sipush			将短整型的常量值（Short.MIN_VALUE ～Short.MAX_VALUE，即 -32768～32767）推送至栈顶
0x12 		ldc 			将 int、float 或 String 型常量值从常量池中推送至栈顶
0x13 		ldc_w			将 int、float 或 String 型常量值从常量池中推送至栈顶（宽索引）
0x14 		ldc2_w			将 long 或 double 型常量值从常量池中推送至栈顶（宽索引）
0x15 		iload 			将指定的 int 型局部变量推送至栈顶
0x16 		lload 			将指定的 long 型局部变量推送至栈顶
0x17 		fload 			将指定的 float 型局部变量推送至栈顶
0x18 		dload 			将指定的 double 型局部变量推送至栈顶
0x19 		aload 			将指定的 引用 型局部变量推送至栈顶
0x1a 		iload_0 		将第一个 int 型局部变量推送至栈顶
0x1b 		iload_1 		将第二个 int 型局部变量推送至栈顶
0x1c 		iload_2 		将第三个 int 型局部变量推送至栈顶
0x1d 		iload_3 		将第四个 int 型局部变量推送至栈顶
0x1e 		lload_0 		将第一个 long 型局部变量推送至栈顶
0x1f 		lload_1 		将第二个 long 型局部变量推送至栈顶
0x20 		lload_2 		将第三个 long 型局部变量推送至栈顶
0x21 		lload_3 		将第四个 long 型局部变量推送至栈顶
0x22 		fload_0 		将第一个 float 型局部变量推送至栈顶
0x23 		fload_1 		将第二个 float 型局部变量推送至栈顶
0x24 		fload_2 		将第三个 float 型局部变量推送至栈顶
0x25 		fload_3 		将第四个 float 型局部变量推送至栈顶
0x26 		dload_0 		将第一个 double 型局部变量推送至栈顶
0x27 		dload_1 		将第二个 double 型局部变量推送至栈顶
0x28 		dload_2 		将第三个 double 型局部变量推送至栈顶
0x29 		dload_3 		将第四个 double 型局部变量推送至栈顶
0x2a 		aload_0 		将第一个 引用 型局部变量推送至栈顶
0x2b 		aload_1 		将第二个 引用 型局部变量推送至栈顶
0x2c 		aload_2 		将第三个 引用 型局部变量推送至栈顶
0x2d 		aload_3 		将第四个 引用 型局部变量推送至栈顶
0x2e 		iaload 			将 int 型数组指定索引的值推送至栈顶
0x2f 		laload 			将 long 型数组指定索引的值推送至栈顶
0x30 		faload 			将 float 型数组指定索引的值推送至栈顶
0x31 		daload 			将 double 型数组指定索引的值推送至栈顶
0x32 		aaload 			将 引用 型数组指定索引的值推送至栈顶
0x33 		baload 			将 boolean 或 byte 型数组指定索引的值推送至栈顶
0x34 		caload 			将 char 型数组指定索引的值推送至栈顶
0x35 		saload 			将 short 型数组指定索引的值推送至栈顶
0x36 		istore 			将栈顶 int 型数值存入指定局部变量
0x37 		lstore 			将栈顶 long 型数值存入指定局部变量
0x38 		fstore 			将栈顶 float 型数值存入指定局部变量
0x39 		dstore 			将栈顶 double 型数值存入指定局部变量
0x3a 		astore 			将栈顶 引用 型数值存入指定局部变量
0x3b 		istore_0 		将栈顶 int 型数值存入第一个局部变量
0x3c 		istore_1 		将栈顶 int 型数值存入第二个局部变量
0x3d 		istore_2 		将栈顶 int 型数值存入第三个局部变量
0x3e 		istore_3 		将栈顶 int 型数值存入第四个局部变量
0x3f 		lstore_0 		将栈顶 long 型数值存入第一个局部变量
0x40 		lstore_1 		将栈顶 long 型数值存入第二个局部变量
0x41 		lstore_2 		将栈顶 long 型数值存入第三个局部变量
0x42 		lstore_3 		将栈顶 long 型数值存入第四个局部变量
0x43 		fstore_0 		将栈顶 float 型数值存入第一个局部变量
0x44 		fstore_1 		将栈顶 float 型数值存入第二个局部变量
0x45 		fstore_2 		将栈顶 float 型数值存入第三个局部变量
0x46 		fstore_3 		将栈顶 float 型数值存入第四个局部变量
0x47 		dstore_0 		将栈顶 double 型数值存入第一个局部变量
0x48 		dstore_1 		将栈顶 double 型数值存入第二个局部变量
0x49 		dstore_2 		将栈顶 double 型数值存入第三个局部变量
0x4a 		dstore_3 		将栈顶 double 型数值存入第四个局部变量
0x4b 		astore_0 		将栈顶 引用 型数值存入第一个局部变量
0x4c 		astore_1 		将栈顶 引用 型数值存入第二个局部变量
0x4d 		astore_2 		将栈顶 引用 型数值存入第三个局部变量
0x4e 		astore_3 		将栈顶 引用 型数值存入第四个局部变量
0x4f 		iastore 		将栈顶 int 型数值存入指定数组的指定索引位置
0x50 		lastore 		将栈顶 long 型数值存入指定数组的指定索引位置
0x51 		fastore 		将栈顶 float 型数值存入指定数组的指定索引位置
0x52 		dastore 		将栈顶 double 型数值存入指定数组的指定索引位置
0x53 		aastore 		将栈顶 引用 型数值存入指定数组的指定索引位置
0x54 		bastore 		将栈顶 boolean 或 byte 型数值存入指定数组的指定索引位置
0x55 		castore 		将栈顶 char 型数值存入指定数组的指定索引位置
0x56 		sastore 		将栈顶 short 型数值存入指定数组的指定索引位置
0x57 		pop 			将栈顶数值弹出（数值不能是 long 或 double 类型的）
0x58 		pop2			将栈顶的一个（对于 long 或 double 类型）或两个数值（对于非 long 或 double 的其他类型）弹出
0x59 		dup 			复制栈顶数值并将复制值压入栈顶
0x5a 		dup_x1 			复制栈顶数值并将两个复制值压入栈顶
0x5b 		dup_x2 			复制栈顶数值并将三个（或两个）复制值压入栈顶
0x5c 		dup2			复制栈顶一个（对于 long 或 double 类型）或两个数值（对于非 long 或 double 的其他类型）并将复制值压入栈顶
0x5d 		dup2_x1 		dup_x1 指令的双倍版本
0x5e 		dup2_x2 		dup_x2 指令的双倍版本
0x5f 		swap	 		将栈最顶端的两个数值互换（数值不能是 long 或 double 类型）
0x60 		iadd 			将栈顶两 int 型数值相加并将结果压入栈顶
0x61 		ladd 			将栈顶两 long 型数值相加并将结果压入栈顶
0x62 		fadd 			将栈顶两 float 型数值相加并将结果压入栈顶
0x63 		dadd 			将栈顶两 double 型数值相加并将结果压入栈顶
0x64 		isub 			将栈顶两 int 型数值相减并将结果压入栈顶
0x65 		lsub 			将栈顶两 long 型数值相减并将结果压入栈顶
0x66 		fsub 			将栈顶两 float 型数值相减并将结果压入栈顶
0x67 		dsub 			将栈顶两 double 型数值相减并将结果压入栈顶
0x68 		imul 			将栈顶两 int 型数值相乘并将结果压入栈顶
0x69 		lmul 			将栈顶两 long 型数值相乘并将结果压入栈顶
0x6a 		fmul 			将栈顶两 float 型数值相乘并将结果压入栈顶
0x6b 		dmul 			将栈顶两 double 型数值相乘并将结果压入栈顶
0x6c 		idiv 			将栈顶两 int 型数值相除并将结果压入栈顶
0x6d 		ldiv 			将栈顶两 long 型数值相除并将结果压入栈顶
0x6e 		fdiv 			将栈顶两 float 型数值相除并将结果压入栈顶
0x6f 		ddiv 			将栈顶两 double 型数值相除并将结果压入栈顶
0x70 		irem 			将栈顶两 int 型数值作取模运算并将结果压入栈顶
0x71 		lrem 			将栈顶两 long 型数值作取模运算并将结果压入栈顶
0x72 		frem 			将栈顶两 float 型数值作取模运算并将结果压入栈顶
0x73 		drem 			将栈顶两 double 型数值作取模运算并将结果压入栈顶
0x74 		ineg 			将栈顶两 int 型数值取负并将结果压入栈顶
0x75 		lneg 			将栈顶两 long 型数值取负并将结果压入栈顶
0x76 		fneg 			将栈顶两 float 型数值取负并将结果压入栈顶
0x77 		dneg 			将栈顶两 double 型数值取负并将结果压入栈顶
0x78 		ishl 			将 int 型数值左移指定位数并将结果压入栈顶
0x79 		lshl 			将 long 型数值左移指定位数并将结果压入栈顶
0x7a 		ishr 			将 int 型数值右（带符号）移指定位数并将结果压入栈顶
0x7b 		lshr 			将 long 型数值右（带符号）移指定位数并将结果压入栈顶
0x7c 		iushr 			将 int 型数值右（无符号）移指定位数并将结果压入栈顶
0x7d 		lushr 			将 long 型数值右（无符号）移指定位数并将结果压入栈顶
0x7e 		iand 			将栈顶两 int 型数值作“按位与”并将结果压入栈顶
0x7f 		land 			将栈顶两 long 型数值作“按位与”并将结果压入栈顶
0x80 		ior 			将栈顶两 int 型数值作“按位或”并将结果压入栈顶
0x81 		lor 			将栈顶两 long 型数值作“按位或”并将结果压入栈顶
0x82 		ixor 			将栈顶两 int 型数值作“按位异或”并将结果压入栈顶
0x83 		lxor 			将栈顶两 long 型数值作“按位异或”并将结果压入栈顶
0x84 		iinc 			M N（M 为非负整数，N 为整数）将局部变量数组的第 M 个单元中的 int 值增加 N，常用于 for 循环中自增量的更新
0x85 		i2l  			将栈顶 int 型数值强制转换成 long 型数值，并将结果压入栈顶
0x86 		i2f  			将栈顶 int 型数值强制转换成 float 型数值，并将结果压入栈顶
0x87 		i2d  			将栈顶 int 型数值强制转换成 double 型数值，并将结果压入栈顶
0x88 		l2i				将栈顶 long 型数值强制转换成 int 型数值，并将结果压入栈顶
0x89 		l2f				将栈顶 long 型数值强制转换成 float 型数值，并将结果压入栈顶
0x8a 		l2d				将栈顶 long 型数值强制转换成 double 型数值，并将结果压入栈顶
0x8b 		f2i				将栈顶 float 型数值强制转换成 int 型数值，并将结果压入栈顶
0x8c 		f2l				将栈顶 float 型数值强制转换成 long 型数值，并将结果压入栈顶
0x8d 		f2d				将栈顶 float 型数值强制转换成 double 型数值，并将结果压入栈顶
0x8e 		d2i				将栈顶 double 型数值强制转换成 int 型数值，并将结果压入栈顶
0x8f 		d2l				将栈顶 double 型数值强制转换成 long 型数值，并将结果压入栈顶
0x90 		d2f				将栈顶 double 型数值强制转换成 float 型数值，并将结果压入栈顶
0x91 		i2b				将栈顶 int 型数值强制转换成 byte 型数值，并将结果压入栈顶
0x92 		i2c				将栈顶 int 型数值强制转换成 char 型数值，并将结果压入栈顶
0x93 		i2s				将栈顶 int 型数值强制转换成 short 型数值，并将结果压入栈顶
0x94 		lcmp			比较栈顶两 long 型数值的大小，并将结果（1、0 或 -1）压入栈顶
0x95 		fcmpl			比较栈顶两 float 型数值的大小，并将结果（1、0 或 -1）压入栈顶 ；当其中一个数值为 “NaN” 时，将 -1 压入栈顶
0x96 		fcmpg			比较栈顶两 float 型数值的大小，并将结果（1、0 或 -1）压入栈顶 ；当其中一个数值为 “NaN” 时，将 1 压入栈顶
0x97 		dcmpl			比较栈顶两 double 型数值的大小，并将结果（1、0 或 -1）压入栈顶 ；当其中一个数值为 “NaN” 时，将 -1 压入栈顶
0x98 		dcmpg			比较栈顶两 double 型数值的大小，并将结果（1、0 或 -1）压入栈顶 ；当其中一个数值为 “NaN” 时，将 1 压入栈顶
0x99 		ifeq 			当栈顶 int 型数值等于 0 时跳转
0x9a 		ifne 			当栈顶 int 型数值不等于 0 时跳转
0x9b 		iflt 			当栈顶 int 型数值小于 0 时跳转
0x9c 		ifge 			当栈顶 int 型数值大于或等于 0 时跳转
0x9d 		ifgt 			当栈顶 int 型数值大于 0 时跳转
0x9e 		ifle 			当栈顶 int 型数值小于或等于 0 时跳转
0x9f 		if_icmpeq 		比较栈顶两 int 型数值的大小，当结果等于 0 时跳转
0xa0 		if_icmpne 		比较栈顶两 int 型数值的大小，当结果不等于 0 时跳转
0xa1 		if_icmplt 		比较栈顶两 int 型数值的大小，当结果小于 0 时跳转
0xa2 		if_icmpge 		比较栈顶两 int 型数值的大小，当结果大于或等于 0 时跳转
0xa3 		if_icmpgt 		比较栈顶两 int 型数值的大小，当结果大于 0 时跳转
0xa4 		if_icmple 		比较栈顶两 int 型数值的大小，当结果小于或等于 0 时跳转
0xa5 		if_acmpeq 		比较栈顶两 引用 型数值，当结果相等时跳转
0xa6 		if_acmpne 		比较栈顶两 引用 型数值，当结果不相等时跳转
0xa7 		goto 			无条件跳转
0xa8 		jsr				跳转至指定的 16 位 offset 位置，并将 jsr 的下一条指令地址压入栈顶
0xa9 		ret				返回至局部变量指定的 index 的指令位置（一般与 jsr 或jsr_w 联合使用）
0xaa 		tableswitch 	用于 switch 条件跳转，case 值连续（可变长度指令）
0xab 		lookupswitch 	用于 switch 条件跳转，case 值不连续（可变长度指令）
0xac 		ireturn 		从当前方法返回 int
0xad 		lreturn 		从当前方法返回 long
0xae 		freturn 		从当前方法返回 float
0xaf 		dreturn 		从当前方法返回 double
0xb0 		areturn 		从当前方法返回对象引用
0xb1 		return 			从当前方法返回 void
0xb2 		getstatic 		获取指定类的静态字段，并将其压入栈顶
0xb3 		putstatic 		为指定类的静态字段赋值
0xb4 		getfield 		获取指定类的实例字段，并将其压入栈顶
0xb5 		putfield 		为指定类的实例字段赋值
0xb6 		invokevirtual 	调用实例方法
0xb7 		invokespecial 	调用超类构造方法，实例初始化方法，私有方法
0xb8 		invokestatic  	调用静态方法
0xb9 		invokeinterface 调用接口方法
0xba 		-- 				无此指令
0xbb 		new 			创建一个对象，并将其引用值压入栈顶
0xbc 		newarray		创建一个指定的原始类型（如 int、float、char 等）的数组，并将其引用值压入栈顶
0xbd 		anewarray		创建一个引用型（如类、接口、数组 等）的数组，并将其引用值压入栈顶
0xbe 		arraylength 	获得数组的长度值并将其压入栈顶
0xbf 		athrow 			将栈顶的异常抛出
0xc0 		checkcast 		校验类型转换，校验未通过将抛出 ClassCastException
0xc1 		instanceof		校验对象是否是指定的类的实例，如果是则将 1 压入栈顶，否则将 0 压入栈顶
0xc2 		monitorenter 	获得对象的锁，用于同步方法或同步块
0xc3 		monitorexit 	释放对象的锁，用于同步方法或同步块
0xc4 		wide 			扩展局部变量的宽度
0xc5 		multianewarray	创建指定类型和指定维度的多维数组（执行该指定时，操作数栈中必须包含各维度的长度），并将其引用值压入栈顶
0xc6 		ifnull 			为 null 时跳转
0xc7 		ifnonnull 		不为 null 时跳转
0xc8 		goto_w 			无条件跳转（宽索引）
0xc9 		jsr_w			跳转至指定的 32 位 offset 位置，并将 jsr_w 的下一条指令地址压入栈顶

```





### 1.2 基本数据类型
1、除了long和double类型外，每个变量都占局部变量区中的一个变量槽(slot)，而long及double会占用两个连续的变量槽。
2、大多数对于boolean、byte、short和char类型数据的操作，都使用相应的int类型作为运算类型。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241424444.png)

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241424301.png)

**一、 加载和存储指令**
1、将一个局部变量加载到操作栈：
```
iload、iload_＜n＞、
lload、lload_＜n＞、
fload、fload_＜n＞、
dload、dload_＜n＞、
aload、aload_＜n＞
```

2、将一个数值从操作数栈存储到局部变量表：
```
istore、istore_＜n＞、
lstore、lstore_＜n＞、
fstore、fstore_＜n＞、
dstore、dstore_＜n＞、
astore、astore_＜n＞
```

3、将一个常量加载到操作数栈：
```
bipush、sipush、
ldc、ldc_w、ldc2_w、
aconst_null、iconst_m1、iconst_＜i＞、lconst_＜l＞、fconst_＜f＞、dconst_＜d＞
```

4、扩充局部变量表的访问索引的指令：
```
wide_＜n＞:_0、_1、_2、_3，
```

存储数据的操作数栈和局部变量表主要就是由加载和存储指令进行操作，除此之外，还有少量指令，如访问对象的字段或数组元素的指令也会向操作数栈传输数据。



**二、 const系列**
该系列命令主要负责把简单的数值类型送到栈顶。该系列命令不带参数。
注意只把简单的数值类型送到栈顶时，才使用如下的命令。
比如对应int型，该方式只能把-1，0，1，2，3，4，5（分别采用iconst_m1，iconst_0，iconst_1， iconst_2， iconst_3， iconst_4， iconst_5）送到栈顶。
对于int型，其他的数值请使用push系列命令（比如bipush）。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241425626.png)

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241425152.png)



**三、 push系列**
该系列命令负责把一个整形数字（长度比较小）送到到栈顶。该系列命令有一个参数，用于指定要送到栈顶的数字。
注意：该系列命令只能操作一定范围内的整形数值，超出该范围的使用将使用ldc命令系列。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241425577.png)



**四、 ldc系列**
该系列命令负责把数值常量或String常量值从常量池中推送至栈顶。该命令后面需要给一个表示常量在常量池中位置(编号)的参数。
哪些常量是放在常量池呢？比如：
```java
final static int id=32768;
final static float double=6.5
```
对于const系列命令和push系列命令操作范围之外的数值类型常量，都放在常量池中。
另外，所有不是通过new创建的String都是放在常量池中的。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241426774.png)



**五、 load系列**
**5.1 load系列A**
该系列命令负责把本地变量的送到栈顶。这里的本地变量不仅可以是数值类型，还可以是引用类型。
  对于前四个本地变量可以采用iload_0，iload_1，iload_2，iload_3(它们分别表示第0，1，2，3个整形变量)这种不带参数的简化命令形式。
  对于第4以上的本地变量将使用iload命令这种形式，在它后面给一参数，以表示是对第几个(从0开始)本类型的本地变量进行操作。对本地变量所进行的编号，是对所有类型的本地变量进行的（并不按照类型分类）。
  对于非静态函数，第一变量是this，即其对应的操作是aload_0。还有函数传入参数也算本地变量，在进行编号时，它是先于函数体的本地变量的。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241426538.png)

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241426568.png)

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241427009.png)






**5.2 load系列B**
该系列命令负责把数组的某项送到栈顶。该命令根据栈里内容来确定对哪个数组的哪项进行操作。
  比如，如果有成员变量：
```
  final String names[]={"robin"，"hb"};
```
那么这句话：
```
String str=names[0];
```
对应的指令为
```
17: aload_0	//将this引用推送至栈顶，即压入栈。

18: getfield #5; 	//Field names:[Ljava/lang/String;
    //将栈顶的指定的对象的第5个实例域（Field）的值（这个值可能是引用，这里就是引用）
压入栈顶

21: iconst_0	//数组的索引值（下标）推至栈顶，即压入栈

22: aaload	//根据栈里内容来把name数组的第一项的值推至栈顶

23: astore 5	//把栈顶的值存到str变量里。因为str在我的程序中是其所在非静态函数的
第5个变量(从0开始计数)，
```

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241427019.png)

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241427734.png)




**六、 store系列**
**6. 1 store系列A**
该系列命令负责把栈顶的值存入本地变量。这里的本地变量不仅可以是数值类型，还可以是引用类型。
如果是把栈顶的值存入到前四个本地变量的话，采用的是istore_0，istore_1，istore_2，istore_3(它们分别表示第0，1，2，3个本地整形变量)这种不带参数的简化命令形式。

如果是把栈顶的值存入到第四个以上本地变量的话，将使用istore命令这种形式，在它后面给一参数，以表示是把栈顶的值存入到第几个(从0开始)本地变量中。对本地变量所进行的编号，是对所有类型的本地变量进行的（并不按照类型分类）。

对于非静态函数，第一变量是this，它是只读的.

还有函数传入参数也算本地变量，在进行编号时，它是先于函数体的本地变量的。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241428678.png)

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241428385.png)

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241428730.png)




**6.2  store系列B**
该系列命令负责把栈顶项的值存到数组里。该命令根据栈里内容来确定对哪个数组的哪项进行操作。
  比如，如下代码:
```java
  int moneys[]=new int[5];
  moneys[1]=100;
```
其对应的指令为：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241429835.png)

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241429145.png)

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241429851.png)





**七、 pop系列**
该系列命令似乎只是简单对栈顶进行操作，更多详情待补充。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241429999.png)




**八、 栈顶元素数学操作及移位操作系列**
该系列命令用于对栈顶元素行数学操作，和对数值进行移位操作。移位操作的操作数和要移位的数都是从栈里取得。
比如对于代码：
```java
int k=100;k=k>>1;
```
其对应的JVM指令为：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241430126.png)

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241430502.png)

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241430308.png)

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241430052.png)

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241431767.png)



**运算指令**
1、运算或算术指令用于对两个操作数栈上的值进行某种特定运算，并把结果重新存入到操作栈顶。
2、算术指令分为两种：整型运算的指令和浮点型运算的指令。
3、无论是哪种算术指令，都使用Java虚拟机的数据类型，由于没有直接支持byte、short、char和boolean类型的算术指令，使用操作int类型的指令代替。
```
加法指令：iadd、ladd、fadd、dadd。 
减法指令：isub、lsub、fsub、dsub。 
乘法指令：imul、lmul、fmul、dmul。 
除法指令：idiv、ldiv、fdiv、ddiv。 
求余指令：irem、lrem、frem、drem。 
取反指令：ineg、lneg、fneg、dneg。 
位移指令：ishl、ishr、iushr、lshl、lshr、lushr。
按位或指令：ior、lor。
按位与指令：iand、land。 
按位异或指令：ixor、lxor。 
局部变量自增指令：iinc。 
比较指令：dcmpg、dcmpl、fcmpg、fcmpl、lcmp。
```

**类型转换指令**
1、类型转换指令可以将两种不同的数值类型进行相互转换。
2、这些转换操作一般用于实现用户代码中的显式类型转换操作，或者用来处理字节码指令集中数据类型相关指令无法与数据类型一一对应的问题。

**宽化类型转换**
int类型到long、float或者double类型。
long类型到float、double类型。
float类型到double类型。
```
i2l、f2b、l2f、l2d、f2d。
```

**窄化类型转换**
```
i2b、i2c、i2s、l2i、f2i、f2l、d2i、d2l和d2f。
```

**对象创建与访问指令**
创建类实例的指令：new。
创建数组的指令：newarray、anewarray、multianewarray。
访问类字段（static字段，或者称为类变量）和实例字段（非static字段，或者称为实例变量）的指令：getfield、putfield、getstatic、putstatic。
把一个数组元素加载到操作数栈的指令：baload、caload、saload、iaload、laload、faload、daload、aaload。
将一个操作数栈的值存储到数组元素中的指令：bastore、castore、sastore、iastore、fastore、dastore、aastore。
取数组长度的指令：arraylength。
检查类实例类型的指令：instanceof、checkcast。


**操作数栈管理指令**
直接操作操作数栈的指令：
将操作数栈的栈顶一个或两个元素出栈：pop、pop2。
复制栈顶一个或两个数值并将复制值或双份的复制值重新压入栈顶：dup、dup2、dup_x1、dup2_x1、dup_x2、dup2_x2。
将栈最顶端的两个数值互换：swap。


**控制转移指令**
1、控制转移指令可以让Java虚拟机有条件或无条件地从指定的位置指令而不是控制转移指令的下一条指令继续执行程序。
2、从概念模型上理解，可以认为控制转移指令就是在有条件或无条件地修改PC寄存器的值。

条件分支：ifeq、iflt、ifle、ifne、ifgt、ifge、ifnull、ifnonnull、if_icmpeq、if_icmpne、if_icmplt、if_icmpgt、if_icmple、if_icmpge、if_acmpeq和if_acmpne。
复合条件分支：tableswitch、lookupswitch。
无条件分支：goto、goto_w、jsr、jsr_w、ret。

在Java虚拟机中有专门的指令集用来处理int和reference类型的条件分支比较操作，为了可以无须明显标识一个实体值是否null，也有专门的指令用来检测null值。










## 2. 方法执行
以下面代码为例看一下执行引擎是如何将一段代码在执行部件上执行的，如下一段代码：
```java
  public  class Math{
    public static void main(String[] args){ 
          int a = 1 ;
          int b = 2;
          int c = (a+b)*10;
    }
  }
```
其中main的字节码指令如下：
```
偏移量       指令                  说明
0：           iconst_1            常数1入栈
1：           istore_1             将栈顶元素移入本地变量1存储 
2：           iconst_2             常数2入栈
3：           istore_2             将栈顶元素移入本地变量2存储 
4：           iload_1              本地变量1入栈
5：           iload_2              本地变量2入栈
6：           iadd                弹出栈顶两个元素相加
7：           bipush 10           将10入栈
9：           imul                  栈顶两个元素相乘
10：         istore_3             栈顶元素移入本地变量3存储 
11：         return                返回
```
对应到执行引擎的各执行部件如图：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241431511.png)

在开始执行方法之前，PC寄存器存储的指针是第1条指令的地址，局部变量区和操作栈都没有数据。从第1条到第4条指令分别将a、b两个本地变量赋值，对应到局部变量区就是1和2分别存储常数1和2，如图：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241432035.png)

第5条和第6条指令分别是将两个局部变量入栈，然后相加，如图：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241433623.png)

1先入栈2后入栈，栈顶元素是2，第7条指令是将栈顶的两个元素弹出后相加，结果再入栈，如图：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241433008.png)

可以看出，变量a和b相加的结果3存在当前栈顶中，接下来第8条指令将10入栈，如图：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241433726.png)

当前PC寄存器执行的地址是9，下一个操作是将当前栈的两个操作数弹出进行相乘并把结果压入栈中，如图：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241433814.png)

第10条指令是将当前的栈顶元素存入局部变量3中，如图：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206241434190.png)

第10条指令执行完后栈中元素出栈，出栈的元素存储在局部变量区3中，对应的是变量c的值。最后一条指令是return，这条指令执行完后当前的这个方法对应的这些部件会被JVM回收，局部变量区的所有值将全部释放，PC寄存器会被销毁，在Java栈中与这个方法对应的栈帧将消失。









## 3. 方法调用
### 3.1 重载和重写
  同一个类中，如果出现多个名称相同，并且参数类型相同的方法，将无法通过编译。因此，想要在同一个类中定义名字相同的方法，那么它们的参数类型必须不同。这种方法上的联系就是重载。

重载的方法在编译过程中即可完成识别。具体到每一个方法调用，Java编译器会根据所传入参数的声明类型(有别实际类型)来选取重载方法。


**选取过程如下：**
1. 不考虑对基本类型自动装拆箱(auto-boxing，auto-unboxing)，以及可变长参数的情况下选取重载方法;
2. 如果1中未找到适配的方法，则允许自动装拆箱，但不允许可变长参数的情况下选取重载方法;
3. 如果2中未找到适配的方法，则在允许自动装拆箱以及可变长参数的情况下选取重载方法.
     那么，如果子类定义了与父类中非私有方法同名的方法，而且这两个方法的参数类型相同，那么这两个方法之间又是什么关系呢？
       如果这两个方法都是静态的，那么子类中的方法隐藏了父类中的方法。
       如果这两个方法都不是静态的，且都不是私有的，那么子类的方法重写了父类中的方法。

众所周知，Java 是一门面向对象的编程语言，它的一个重要特性便是多态。而方法重写，正是多态最重要的一种体现方式：它允许子类在继承父类部分功能的同时，拥有自己独特的行为。




### 3.2 JVM的静态绑定和动态绑定
Java虚拟机识别方法的关键在于【类名 + 方法名 + 方法描述符(method descriptor)】。
注：方法描述符由方法的【参数类型 + 返回型】构成。
把一个【方法】与其所在的【类/对象】关联起来叫做方法的绑定。方法绑定分为静态绑定（前期绑定）和动态绑定（后期绑定）。


**静态绑定**
Java虚拟机中的静态绑定(static binding)指的是在解析时便能够直接识别目标方法的情况；在程序运行前就已经知道方法是属于那个类的，在编译的时候就可以连接到类的中，定位到这个方法。

在Java中，final、private、static修饰的方法以及构造函数都是静态绑定的，不需程序运行，不需具体的实例对象就可以知道这个方法的具体内容。


**动态绑定**
而动态绑定(dynamic binding)则指的是需要在运行过程中根据调用者的动态类型（具体的实例对象）来识别目标方法的情况。

动态绑定是多态性得以实现的重要因素，它通过方法表来实现：每个类被加载到虚拟机时，在方法区保存元数据，其中，包括一个叫做 方法表（method table）的东西，表中记录了这个类定义的方法的指针，每个表项指向一个具体的方法代码。如果这个类重写了父类中的某个方法，则对应表项指向新的代码实现处。从父类继承来的方法位于子类定义的方法的前面。

动态绑定语句的编译、 运行原理：我们假设Son继承自Father，重写了say()。
```java
Father ft = new Son();
ft.say();
```

1：编译：我们知道，向上转型时，用父类引用指向子类对象，并可以用父类引用调用子类中重写了的同名方法。但是不能调用子类中新增的方法，为什么呢？
因为在代码的编译阶段，编译器通过 声明对象的类型（ 即引用本身的类型） 在方法区中该类型的方法表中查找匹配的方法（最佳匹配法：参数类型最接近的被调用），如果有则编译通过。（这里是根据声明的对象类型来查找的，所以此处是查找 Father类的方法表，而Father类方法表中是没有子类新增的方法的，所以不能调用。）

编译阶段是确保方法的存在性，保证程序能顺利、安全运行。

2：运行：我们又知道，ft.say()调用的是Son中的say()，这不就与上面说的，查找Father类的方法表的匹配方法矛盾了吗？不，这里就是动态绑定机制的真正体现。

上面编译阶段在 声明对象类型 的方法表中查找方法，只是为了安全地通过编译（ 也为了检验方法是否是存在的） 。而在实际运行这条语句时，在执行 Father ft=new Son(); 这一句时创建了一个Son实例对象，然后在 ft.say() 调用方法时，JVM会把刚才的son对象压入操作数栈，用它来进行调用。而用实例对象进行方法调用的过程就是动态绑定：根据实例对象所属的类型去查找它的方法表， 找到匹配的方法进行调用。 我们知道，子类中如果重写了父类的方法，则方法表中同名表项会指向子类的方法代码；若无重写，则按照父类中的方法表顺序保存在子类方法表中。

故此：动态绑定根据对象的类型的方法表查找方法是一定会匹配（因为编译时在父类方法表中以及查找并匹配成功了，说明方法是存在的。这也解释了为何向上转型时父类引用不能调用子类新增的方法：在父类方法表中必须先对这个方法的存在性进行检验， 如果在运行时才检验就容易出危险——可能子类中也没有这个方法）。

程序在JVM运行过程中，会把类的类型信息、static属性和方法、final常量等元数据加载到方法区，这些在类被加载时就已经知道， 不需对象的创建就能访问的， 就是静态绑定的内容； 需要等对象创建出来， 使用时根据堆中的实例对象的类型才进行取用的就是动态绑定的内容。
我们再从JVM层面分析下，JVM里面是通过哪里指令来实现方法的调用的：
```
- invokestatic:调用静态方法
- invokeinterface:调用接口方法([多态]())
- invokespecial:调用非静态私有方法、构造方法(包括super)
- invokevirtual:调用非静态非私有方法([多态]())
- invokedynamic:动态调用（Java7引入的，第一次用却是在Java8中，用在了Lambda表达式和默认方法中，它允许调用任意类中的同名方法，注意是任意类，和重载重写不同）(动态 ≠ 多态)
```
那么这些指令又是怎么来调用方法的呢？(invokedynamic和这些有点不一样，稍后单独解释下)

在编译的过程中，JVM并不知道目标方法的具体内存地址，此时编译器会用”符号引用”来表示该方法(加载阶段)。当JVM进行到“解析”阶段的时候，这些引用会被替换为直接引用，这个时候就知道需要去哪里调用到方法了！

对于静态绑定的方法，直接引用就是直接指向方法的指针，而对于动态绑定的方法，直接引用其实指向方法表中的一个索引。

方法表是一个数组，每个数组元素指向一个当前类及其父类中非private的实例方法，样子如下所示：
由于动态绑定相比于静态绑定，在寻找方法时要出多好一个内存解析的动作，例如获取调用者类型，获取方法表，获取方法表的索引值等等，还是有点开销的，虽然这些开销是必须的。所以JVM中引入了一些优化的技术： 内存缓联+方法内联。

具体来说，Java字节码指令中与调用相关的指令共有五种：
1) invokevirtual ：
用于调用对象的实例方法，根据对象的实际类型进行分派（虚方法分派），这也是Java语言中最常见的方法分派方式。
2) invokeinterface ：
用于调用接口方法，它会在运行时搜索一个实现了这个接口方法的对象，找出适合的方法进行调用。
3) invokespecial ：
用于调用一些需要特殊处理的实例方法，包括实例初始化（＜init＞）方法、私有方法和父类方法。
4) invokestatic ：
调用静态方法（static方法）。
5) invokedynamic ：
用于在运行时动态解析出调用点限定符所引用的方法，并执行该方法，前面4条调用指令的分派逻辑都固化在Java虚拟机内部，而invokedynamic指令的分派逻辑是由用户所设定的引导方法决定的。
```java
interface Student {
    boolean isRecommend();
}
class Edu {
    public double youhui (double originPrice， Student stu) {
        return originPrice * 0.7d;
    }
}

class Kaikeba extends Edu {
  @Override
  public double youhui (double originPrice， Student stu) {
    if (stu.isRecommend()) {                    // invokeinterface 
      return originPrice * randomYouhui ();     // invokestatic
    } else {
      return super.youhui(originPrice， stu);   // invokespecial
    }
  }
  private static double randomYouhui () {
    return new Random()                          // invokespecial 
           .nextDouble()   ;                     // invokevirtual
  }
}
```



### 3.3 调用指令的符号引用

在编译过程中，目标方法的具体内存地址尚未确定.这时，Java编译器会暂时用符号引用来表示该目标方法.这一符号引用包括目标方法所在的类或接口的名字，以及目标方法的方法名和方法描述符.

符号引用存储在class文件的常量池中.根据目标方法是否为接口方法，又可分为接口符号引用和非接口符号引用。

对于非接口符号引用，假定该目标方法的符号引用所指向的类为 C，则 Java 虚拟机会按照如下步骤进行查找。
1. 在 C 中查找符合名字及描述符的方法（本类中查找）。
2. 如果没有找到，在 C 的父类中继续搜索，直至 Object 类（父类中查找）。
3. 如果没有找到，在 C 所直接实现或间接实现的接口搜索，这一步搜索得到的目标方法必须是非私有、非静态的。并且，如果目标方法在间接实现的接口中，则需满足 C 与该接口之间没有其他符合条件的目标方法。如果有多个符合条件的目标方法，则任意返回其中一个（接口中查找）。

从这个解析算法可以看出，静态方法也可以通过子类来调用。此外，子类的静态方法会隐藏（注意与重写区分）父类中的同名、同描述符的静态方法。

对于接口符号引用，假定该目标方法的符号引用所指向的接口为 I，则 Java 虚拟机会按照如下步骤进行查找。
1. 在 I 中查找符合名字及描述符的方法（本接口查找）。
2. 如果没有找到，在 Object 类中的公有实例方法中搜索（Object类查找）。
3. 如果没有找到，则在 I 的超接口中搜索。这一步的搜索结果的要求与非接口符号引用步骤 3 的要求一致（父类接口中查找）。

经过上述的解析步骤之后，符号引用会被解析成实际引用。对于可以静态绑定的方法调用而言，实际引用是一个指向方法的指针。对于需要动态绑定的方法调用而言，实际引用则是一个方法表的索引。



### 3.4 虚方法调用

所有非私有实例方法被调用-->编译-->invokevirtual指令.

接口方法调用-->编译-->invokeinterface指令.

这两种指令，均属于Java虚拟机中的虚方法调用.

多数情况下，Java虚拟机需要根据调用者的动态类型-->确定虚方法调用的目标方法。这个过程被称为动态绑定.
相对于静态绑定的非虚方法调用，虚方法调用更加耗时.

在Java虚拟机中，静态绑定包括用于调用静态方法的invokestatic指令，和用于调用构造器/私有（private）实例方法/超类非私有实例方法的invokespecial指令.

如果虚方法调用指向一个标记为final的方法，那么Java虚拟机也可以静态绑定该虚方法调用的目标方法.

Java虚拟机采用了一种用空间换时间的策略来实现动态绑定.它为每个类生成一张方法表，用以快速定位目标方法.



### 3.5 方法表

类加载的准备阶段，除了为静态字段分配内存外，还会构建与该类相关联的方法表.
方法表，是Java虚拟机实现动态绑定的关键所在.
方法表本质上是一个数组，每个数组元素指向一个当前类及其父类中非私有的实例方法.

**方法表满足两个特质:**
1. 子类方法表中包含父类方法表中的所有方法
2. 子类方法在方法表中的索引值，与它所重写的父类方法的索引值相同.
     方法调用指令中的符号引用会在执行之前解析为实际引用.

**静态绑定的方法调用:**实际引用-->具体的目标方法。
**动态绑定的方法调用:**实际引用-->方法表的索引值(实际上不止索引值)。在执行过程中，Java虚拟机将获取调用者的实际类型，并在该实际类型的虚方法表中，根据索引值获得目标方法--->动态绑定的过程。

事实上，使用了方法表的动态绑定与静态绑定相比，仅仅多出几个内存解引用操作 ： 访问栈上的调用者，读取调用者的动态类型，读取该类型的方法表，读取方法表中某个索引值所对应的目标方法。相对于创建并初始化Java栈帧来说，这几个内存解引用操作的开销可以忽略不计。

但是，虚方法调用对性能仍有影响：
方法表的引入带来的优化效果仅存在于解释执行或者即时编译代码的最坏情况下。而且即时编译还拥有两个性能更好的优化手段：内联缓存(inlining cache)和方法内联(method inlining)。



### 3.6 内联缓存

内联缓存是一种加快动态绑定的优化技术。它能够缓存虚方法调用中调用者的动态类型，以及该类型所对应的目标方法。后续执行中，优先使用缓存，没有则使用基于方法表的动态绑定。

对多态的优化术语：
1. 单态(monomorphic)，指的是仅有一种状态的情况。
2. 多态(polymorphic)，指的是有限数量种状态的情况。二态(bimorphic)是多态的其中一种。
3. 超多态(megamorphic)，指的是更多种状态的情况。通常用某个阈值来区分多态和超多态。

综上，内联缓存对应单态内联缓存/多态内联缓存/超多态内联缓存。
1. 单态内联缓存，顾名思义，便是只缓存了一种动态类型以及它所对应的目标方法。它的实现非常简单：比较所缓存的动态类型，如果命中，则直接调用对应的目标方法。
2. 多态内联缓存则缓存了多个动态类型及其目标方法。它需要逐个将所缓存的动态类型与当前动态类型进行比较，如果命中，则调用对应的目标方法。
     注：一般来说，我们会将更加热门的动态类型放在前面。在实践中，大部分的虚方法调用均是单态的，也就是只有一种动态类型。为了节省内存空间，Java 虚拟机只采用单态内联缓存。

在选择内联缓存时，如果未命中则重新使用方法表做动态绑定。这时有两种选择：
1. 替换单态内联缓存中的纪录。这种做法就好比 CPU 中的数据缓存，它对数据的局部性有要求，即在替换内联缓存之后的一段时间内，方法调用的调用者的动态类型应当保持一致，从而能够有效地利用内联缓存。因此，在最坏情况下，用两种不同类型的调用者，轮流执行该方法调用，那么每次进行方法调用都将替换内联缓存。也就是说，只有写缓存的额外开销，而没有使用缓存的性能提升。

2. 劣化为超多态状态。这也是 Java 虚拟机的具体实现方式。处于这种状态下的内联缓存，实际上放弃了优化的机会。它将直接访问方法表，来动态绑定目标方法。与替换内联缓存纪录的做法相比，它牺牲了优化的机会，但是节省了写缓存的额外开销。

虽然内联缓存附带内联二字，但是它并没有内联目标方法。这里需要明确的是，任何方法调用除非被内联，否则都会有固定开销。这些开销来源于保存程序在该方法中的执行位置，以及新建、压入和弹出新方法所使用的栈帧。



### 3.7 JVM处理invokedynamic

在Java中，方法调用会编译为invokestatic/invokespecial/invokevirtual/invokeinterface四种指令。这些类名与包含目标方法类名/方法名/方法描述符的符号引用捆绑。在实际运行之前，Java虚拟机将根据这个符号引用链接到具体的目标方法。

Java7引入了invokedynamic指令，该指令的调用机制抽象出调用点这一概念，并允许应用程序将调用点链接至任何符合条件的方法上。

作为invokedynamic的准备工作，Java7引入了更加底层/更加灵活的方法抽象：方法句柄(MethodHandle).



### 3.8 方法句柄的概念

方法句柄是一种强类型的，能够被直接执行的引用。该引用可以指向常规的静态方法或者实例方法，也可以指向构造器或者字段。当指向字段时，方法句柄实则指向包含字段访问字节码的虚构方法，语义上等价于目标字段的getter或者setter方法。

HotSpot虚拟机中方法句柄调用的具体实现 ：
以DirectMethodHandle为例，调用方法句柄所使用的invokeExact或者invoke方法具备签名多态性的特性。会根据具体的传入参数来生成方法描述符。其中，invokeExact要求传入的参数和所指向方法的描述符严格匹配。方法句柄还支持增删改参数的操作，这些操作是通过生成另一个充当适配器的方法句柄来实现的。

方法句柄的调用和反射调用一样，都是间接调用。同样都面临无法内联的问题，不过与反射调用不同的是，方法句柄的内联瓶颈在于即时编译器能否将该方法句柄识别为常量。



### 3.9 invokedynamic指令

invokedynamic是Java7引入的一条新指令，用以支持动态语言的方法调用。具体来说，它将调用点(CallSite)抽象成一个Java类，并且将原本由Java虚拟机控制的方法调用以及方法链接暴露给了应用程序。在运行过程中，每一条invokedynamic指令将捆绑一个调用点，并会调用该调用点所链接的方法句柄.

在第一次执行invokedynamic指令时，Java虚拟机会调用该指令所对应的启动方法(BootStrapMethod)，来生成调用点，并将之绑定至该invokedynamic指令中。在之后的运行过程中，Java虚拟机则会直接调用绑定的调用点所链接的方法句柄。

在字节码中，启动方法是用方法句柄来指定的。这个方法句柄指向一个返回类型为调用点的静态方法。该方法必须接收三个固定的参数，分别为一个Lookup类实例，一个用来指代目标方法名字的字符串，以及该调用点能够链接的方法句柄的类型。

除了三个必须参数外，启动方法(BootStrapMethod)还可以接收若干个其它的参数，用来辅助生成调用点，或者定位索要链接的目标方法。



### 3.10 Java8的Lambda表达式

在Java8中，Lambda表达式也是借助invokedynamic来实现的。

具体来说，Java编译器利用invokedynamic指令来生成实现了函数式接口的适配器。这里的函数式接口指的是仅包括一个非default接口方法的接口，一般通过@FunctionalInterface注解。同时，该invokedynamic指令对应的启动方法将通过ASM生成一个适配器类。

对于没有捕获其它变量的Lambda表达式，该invokedynamic指令始终返回同一个适配器类的实例。对于捕获了其它变量的Lambda表达式，每次执行invokedynamic指令将新建一个适配器类实例。

不管是捕获型的还是未捕获型的Lambda表达式，它们的性能上限皆可以达到直接调用的性能。其中，捕获型Lambda表达式借助了即时编译器的逃逸分析，来避免实际的新建适配器类实例的操作。



### 3.11 总结

1、Class文件的编译过程中不包含传统编译中的连接步骤，所有方法调用中的目标方法在Class文件里 面都是一个常量池中的符号引用，而不是方法在实际运行时内存布局中的入口地址。

2、在类加载的解析阶段，会将其中的一部分符号引用转化为直接引用，这类方法（编译期可知，运行 期不可变）的调用称为解析（Resolution）。

主要包括静态方法和私有方法两大类，前者与类型直接关联，后者在外部不可被访问，这两种方法各自 的特点决定了它们都不可能通过继承或别的方式重写其他版本，因此它们都适合在类加载阶段进行解 析。

3、只要能被invokestatic和invokespecial指令调用的方法，都可以在解析阶段中确定唯一的调用版 本，符合这个条件的有静态方法、私有方法、实例构造器、父类方法4类，它们在类加载的时候就会把 符号引用解析为该方法的直接引用。


