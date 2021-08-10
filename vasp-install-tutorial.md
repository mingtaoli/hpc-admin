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

修改其中的FC为mpif90，加入
```shell
MKLROOT    = /opt/intel/mkl
BLAS       = ${MKLROOT}/lib/intel64/libmkl_scalapack_lp64.a -Wl,--start-group ${MKLROOT}/lib/intel64/libmkl_cdft_core.a ${MKLROOT}/lib/intel64/libmkl_intel_lp64.a ${MKLROOT}/lib/intel64/libmkl_sequential.a ${MKLROOT}/lib/intel64/libmkl_core.a ${MKLROOT}/lib/intel64/libmkl_blacs_openmpi_lp64.a -Wl,--end-group -lpthread -lm -ldl
```
注释掉MKL_PATH，置空BLACS，SCALAPACK，因为最后起作用的是

```shell
LLIBS      = $(SCALAPACK) $(LAPACK) $(BLAS)
```
而我们把所有的这些都放到BLAS了。（按照intel mkl link line advisor得到，参考 https://software.intel.com/en-us/articles/intel-mkl-link-line-advisor/ ）。 INCS变量中加入-I$(MKLROOT)/include/fftw

INCS       =-I$(MKLROOT)/include/fftw -I${MKLROOT}/include



完成后，进行安装：
```shell
make all
```
在`build`目录下生成`std`、`ncl`、`gma`三个可执行文件，安装完成。

如果是编译CUDA版本，修改CUDA为

CUDA_ROOT  = /opt/cuda-12.0/
修改MPI_INC为

MPI_INC    =/openmpi/installed/path/include
然后

make gpu
就得到vasp_gpu了。
我们可以建立一个vasp测试目录， mpirun -np 4 /path/to/vasp_gpu 来看看运行得怎么样。

