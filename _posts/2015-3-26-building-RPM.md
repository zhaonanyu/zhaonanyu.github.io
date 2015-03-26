---
layout: post
title: "制作RPM包"
description: "every geek wants a blog all by himself lol"
tags: [GitHub Pages, jekyll, static site generator]
comments: true
---

标签: RPM RedHat .spec

---

##关于RPM
**RPM包管理员**
RPM包管理员（简称RPM，全称为The RPM Package Manager）是在Linux下广泛使用的软件包管理器。RPM此名词可能是指.rpm的文件格式的软件包，也可能是指其本身的软件包管理器(RPM Package Manager)。最早由Red Hat研制，现在也由开源社区开发。RPM通常随附于Linux发行版，但也有单独将RPM作为应用软件发行的发行版（例如Gentoo）。RPM仅适用于安装用RPM来打包的软件，目前是GNU/Linux下软件包资源最丰富的软件包类型之一。
**RPM软件包**
RPM软件包分为二进制包（Binary）、源代码包（Source）和Delta包三种。二进制包可以直接安装在计算机中，而源代码包将会由RPM自动编译、安装。源代码包经常以src.rpm作为后缀名。

---

##关于SPEC
spec（配置规范文件）。RPM 编译过程的核心是处理 .spec 文件。**它说明了软件包怎样被配置，补缀哪些补丁，安装哪些文件，被安装到哪里，在安装该包之前或之后需要运行那些系统级别的活动。**它必须手写，但更简单的办法是拿来他人写好的，在此基础上修改。RPM 自身对于你能在 spec 文件中做什么没有太多限制。

### 1.宏&变量
在spec文件中存在宏和变量，其本身提供一些宏与变量，他们有对应关系。当然你也可以自己定义。
例子：

|名称|宏风格     |变量风格   |
|:-:      |:-:      | :-:      |
|编译时的根目录| %buildroot | $RPM_BUILD_ROOT |
|优化参数| %optflags | $RPM_OP_FLAGS |

### 2.关键字
spec脚本开头部分有很多关键字，用来提供RPM包本身的信息，有些关键字可以不用填写
**Name:** 软件包的名称，后面可使用%{name}的方式引用

**Summary:** 软件包的内容概要

**Version:** 软件的实际版本号，例如：1.0.1等，后面可使用%{version}引用

**Release:** 发布序列号，例如：1linuxing等，标明第几次打包，后面可使用%{release}引用

**Group:** 软件分组，建议使用标准分组

**License:** 软件授权方式，通常就是GPL

**Source:** 源代码包，可以带多个用Source1、Source2等源，后面也可以用%{source1}、%{source2}引用

**BuildRoot:** 这个是安装或编译时使用的“虚拟目录”，考虑到多用户的环境，一般定义为：
`%{_tmppath}/%{name}-%{version}-%{release}-root`
或
`%{_tmppath}/%{name}-%{version}-%{release}-buildroot-%(%{__id_u} -n}`
该参数非常重要，因为在生成rpm的过程中，执行make install时就会把软件安装到上述的路径中，在打包的时候，同样依赖“虚拟目录”为“根目录”进行操作。
**后面可使用$RPM_BUILD_ROOT 方式引用**。

**URL:** 软件的主页

**Vendor:** 发行商或打包组织的信息，例如RedFlag Co,Ltd

**Disstribution:** 发行版标识

**Patch:** 补丁源码，可使用Patch1、Patch2等标识多个补丁，使用%patch0或%{patch0}引用

**Prefix:** %{_prefix} 这个主要是为了解决今后安装rpm包时，并不一定把软件安装到rpm中打包的目录的情况。这样，必须在这里定义该标识，并在编写%install脚本的时候引用，才能实现rpm安装时重新指定位置的功能

Prefix: %{_sysconfdir} 这个原因和上面的一样，但由于%{_prefix}指/usr，而对于其他的文件，例如/etc下的配置文件，则需要用%{_sysconfdir}标识

**Build Arch:** 指编译的目标处理器架构，noarch标识不指定，但通常都是以/usr/lib/rpm/marcros中的内容为默认值

**Requires:** 该rpm包所依赖的软件包名称，可以用>=或<=表示大于或小于某一特定版本，例如：
libpng-devel >= 1.0.20 zlib 
※“>=”号两边需用空格隔开，而不同软件名称也用空格分开
还有例如PreReq、Requires(pre)、Requires(post)、Requires(preun)、Requires(postun)、BuildRequires等都是针对不同阶段的依赖指定 

**Provides:** 指明本软件一些特定的功能，以便其他rpm识别

**Packager:** 打包者的信息

**%description** 软件的详细说明

### 3.spec脚本本体
**%pre**：预处理脚本
**%build**：开始构建包，在/usr/src/asianux/BUILD/%{name}-%{version}目录中进行make的工作
**%install**：将软件安装至虚拟根目录中，其中`%makeinstall`相当于`make DESTDIR=$RPM_BUILD_ROOT install`。需要说明的是，这里的%install主要就是为了后面的%file服务的。所以，还可以使用常规的系统命令：

~~~shell
install -d $RPM_BUILD_ROOT/
cp -a * $RPM_BUILD_ROOT/
~~~

**%clean**：清理临时文件。通常内容为：

~~~shell
[ "$RPM_BUILD_ROOT" != "/" ] && rm -rf "$RPM_BUILD_ROOT"
rm -rf $RPM_BUILD_DIR/%{name}-%{version}
~~~

**%file**：指定文件及其属性信息。
使用 一次 %defattr 来定义缺省的许可权、所有者和组；在这个示例中， %defattr(-,root,root) 会安装 root 用户拥有的所有文件，使用当 RPM 从构建系统捆绑它们时它们所具有的任何许可权。
可以用 %attr(permissions,user,group) 覆盖个别文件的所有者和许可权。
%defattr (-,root,root) 指定包装文件的属性，分别是(mode,owner,group)，-表示默认值，对文本文件是0644，可执行文件是0755

**%post**：完成安装后续工作
下面的例子中使用了这个标签来完成开机自启脚本等工作

**%changelog**：修改记录

### 4.目录
制作一个RPM包的工作环境rpmbuild中将含有以下目录：
BUILD：编译RPM包时的临时目录%_builddir
BUILDROOT：编译后生成的软件临时安装目录%_buildrootdir
RPMS：最终生成的RPM包成品目录%_rpmdir
SOURCES：所有源代码以及补丁文件的存放目录
SPECS：spec文件的存放目录
SRPMS：软件最终的RPM源码格式存放路径%_srcrpmdir

---

##一个例子
**例子**：kmod.ko放置到指定的目录中，并使其自启动
###步骤
1.安装rpmdev工具

~~~shell
yum install rpmdevtools
~~~

2.创建RPM包工作目录

~~~shell
rpmdev-setuptree
~~~

该命令将会在家目录中生成rpmbuild工作目录
3.创建spec模板

~~~shell
cd ~/rpmbuild/SPECS
rpmdev-newspec xxx-xxx.spec
~~~

生成的spec模板包含spec文件的基本要素
生成结果：

~~~shell
Name:           test
Version:        
Release:        1%{?dist}
Summary:        

Group:          
License:        
URL:            
Source0:        

BuildRequires:  
Requires:       

%description


%prep
%setup -q


%build
%configure
make %{?_smp_mflags}


%install
rm -rf $RPM_BUILD_ROOT
make install DESTDIR=$RPM_BUILD_ROOT


%clean
rm -rf $RPM_BUILD_ROOT


%files
%defattr(-,root,root,-)
%doc



%changelog
~~~

4.将编译完成的kmod.ko模块复制至SOURCES文件夹中，并使用

~~~shell
tar zcvf kmod.tar.gz kmod.ko
rm -rf kmod.ko
~~~

命令生成所需压缩包

5.填写spec文件
例子所使用的spec文件：

~~~shell
Name:           NUMEN
Version:        1.0
Release:        1%{?dist}
Summary:        NUMEN resources

Group:          System
License:        GPL
URL:            zhaonanyu.github.io
Source0:        NUMEN.tar.gz

BuildRoot:      %{_tmppath}/%{name}-%{version}-%{release}-root


%description


%prep
%setup -c

%install
rm -rf $RPM_BUILD_ROOT
install -d $RPM_BUILD_ROOT
cp -a * $RPM_BUILD_ROOT

%clean
rm -rf $RPM_BUILD_ROOT
rm -rf $RPM_BUILD_DIR/%{name}-%{version}

%files
%defattr(-,root,root)
/NUMEN/kmod.ko

%post
if [ -d /lib/modules/`uname -r`/kernel/hello ]; then
        cp -f $RPM_BUILD_ROOT/NUMEN/kmod.ko /lib/modules/`uname -r`/kernel/hello/kmod.ko
else
        mkdir /lib/modules/`uname -r`/kernel/hello
        cp -f $RPM_BUILD_ROOT/NUMEN/kmod.ko /lib/modules/`uname -r`/kernel/hello/kmod.ko
fi

echo "kmod testing self load"
echo "if [ -d /lib/modules/`uname -r`/kernel/hello ]; then" >> /etc/rc.d/rc.sysinit
echo "      depmod kmod > /dev/null 2>&1" >> /etc/rc.d/rc.sysinit
echo "      modprobe kmod > /dev/null 2>&1" >> /etc/rc.d/rc.sysinit
echo "fi" >> /etc/rc.d/rc.sysinit


%changelog

~~~

**关于%prep中的 %setup -c**:
%setup 不加任何选项，仅将软件包打开。 
%setup -n newdir 将软件包解压在newdir目录。 
%setup -c 解压缩之前先产生目录。 
%setup -b num 将第num个source文件解压缩。 
%setup -T 不使用default的解压缩操作。 
%setup -T -b 0 将第0个源代码文件解压缩。 
%setup -c -n newdir 指定目录名称newdir，并在此目录产生rpm套件。 

**%post中的内容解释：**

~~~shell
if [ -d /lib/modules/`uname -r`/kernel/hello ]; then
        cp -f $RPM_BUILD_ROOT/NUMEN/kmod.ko /lib/modules/`uname -r`/kernel/hello/kmod.ko
else
        mkdir /lib/modules/`uname -r`/kernel/hello
        cp -f $RPM_BUILD_ROOT/NUMEN/kmod.ko /lib/modules/`uname -r`/kernel/hello/kmod.ko
fi
~~~

该部分是将kmod.ko放入/lib/modules/`uname -r`/hello目录中，若不存在该目录将会建立这个路径

~~~shell
echo "if [ -d /lib/modules/`uname -r`/kernel/hello ]; then" >> /etc/rc.d/rc.sysinit
echo "      depmod kmod > /dev/null 2>&1" >> /etc/rc.d/rc.sysinit
echo "      modprobe kmod > /dev/null 2>&1" >> /etc/rc.d/rc.sysinit
echo "fi" >> /etc/rc.d/rc.sysinit
~~~

这部分是将modprobe kmod命令通过echo的形式写入/etc/rc.d/rc.sysinit文件中，通过开机脚本来使kmod.ko自启动，注意如果你的kmod.ko并非在本机上编译，你需要使用depmod kmod命令

6.生成RPM包
使用命令：

~~~shell
rpmbuild -ba kmod.spec
~~~

7.测试生成的RPM包
进入RPMS文件夹，找到刚生成的RPM包使用命令

~~~shell
# in super-user
rpm -ivh kmod-x-x.xxx.xxxxx.rpm
~~~

安装完成后重启本机，开机后使用

~~~shell
lsmod | grep kmod
dmesg | grep kmod
~~~

来查看结果