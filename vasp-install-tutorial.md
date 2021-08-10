# 集群上VASP安装记录

下载并解压VASP，这里要安装的是VASP6.2.1版本,安装目录为用户的home目录。
```shell
cp /public/software-packages/vasp/vasp6.2.1.tgz ~/
tar -zxvf vasp6.2.1.tgz
cd vasp6.2.1
tar -zxvf vasp.6.2.1.tgz
cd vasp.6.2.1
```

然后以自带的makefile.include文件为模板，根据实际情况修改具体参数，生成需要的makefile.include文件：
```shell
cp ./arch/makefile.include.linux_intel ./
vi makefile.include.linux_intel
```

完成后，进行安装：
```shell
make all
```

在`build`目录下生成`std`、`ncl`、`gma`三个可执行文件，安装完成