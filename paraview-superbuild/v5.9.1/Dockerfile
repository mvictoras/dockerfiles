FROM centos:centos7

SHELL ["/bin/bash", "-c"]

# env var to preserve path
env ORIGINAL_PATH="$PATH"

##########################
# Install CentOS packages
##########################

run yum update -y 
run yum groupinstall -y "Development Tools"
run yum install -y epel-release
run yum install -y \
    curl \
    gcc gcc-c++ \
    wget \
    git \
    sudo \
    vim \
    automake \
    openssl-devel \
    python3-devel \
    openssh-clients  
run yum autoremove -y

############################
# Build directory structure
############################

# /paraview will contain the final build
# /pvbuild will contain the intermediate files
# and will be deleted at the end to reduce the space
# needed by this image

run mkdir /paraview \
 && mkdir /pvbuild \
 && mkdir /pvbuild/build \
 && mkdir /pvbuild/src \
 && mkdir /pvbuild/install 

#######################################
# CMake
# We need a recent cmake for paraview
#######################################

workdir /pvbuild/install
run wget https://github.com/Kitware/CMake/releases/download/v3.18.4/cmake-3.18.4-Linux-x86_64.tar.gz \
 && tar xvzf cmake-3.18.4-Linux-x86_64.tar.gz \
 && rm cmake-3.18.4-Linux-x86_64.tar.gz \
 && mv cmake-3.18.4-Linux-x86_64 cmake

# add cmake binaries to the path
env PATH="/pvbuild/install/cmake/bin:$PATH"

# gcc 4.8.5 in Centos 7 is too old
# and causes problems
# We build our own GCC 9.2.0

##########################
# GCC 9.2.0
# used instructions from
# from https://gist.github.com/nchaigne/ad06bc867f911a3c0d32939f1e930a11
##########################

workdir /pvbuild/src
run export GCC_VERSION=9.2.0 \
 && wget https://ftp.gnu.org/gnu/gcc/gcc-${GCC_VERSION}/gcc-${GCC_VERSION}.tar.gz \
 && tar xzvf gcc-${GCC_VERSION}.tar.gz \
 && rm gcc-${GCC_VERSION}.tar.gz \
 && cd gcc-${GCC_VERSION} \
 && ./contrib/download_prerequisites \
 && cd /pvbuild/build \
 && mkdir gcc \
 && cd gcc \
 && /pvbuild/src/gcc-${GCC_VERSION}/configure --disable-multilib --enable-languages=c,c++ --prefix=/pvbuild/install/gcc \
 && make -j 8 \
 && make install

# add to local environment
env PATH="/pvbuild/install/gcc/bin:$PATH"
env LD_LIBRARY_PATH="/pvbuild/install/gcc/lib64:$LD_LIBRARY_PATH"

# remove previous versions of gcc because
# they create conflicts with some paraview
# dependencies

run yum remove -y gcc gcc-c++

# but now libtool and libtoolize are also
# removed, so we need to build our own

workdir /pvbuild/src
run wget https://ftpmirror.gnu.org/libtool/libtool-2.4.6.tar.gz \
 && tar xvf libtool-2.4.6.tar.gz \
 && rm libtool-2.4.6.tar.gz
run mkdir /pvbuild/build/libtool
workdir /pvbuild/build/libtool
run /pvbuild/src/libtool-2.4.6/configure --prefix=/pvbuild/install/libtool
run make -j8
run make install

env PATH="/pvbuild/install/libtool/bin:$PATH"
env LD_LIBRARY_PATH="/pvbuild/install/libtool/lib:$LD_LIBRARY_PATH"

#############################
# Mpich 3.2.1
#
# We build Mpich and
# then delete it. At runtime
# we will use the native MPI
# on the host system
#############################

workdir /pvbuild/src
run wget -q http://www.mpich.org/static/downloads/3.2.1/mpich-3.2.1.tar.gz \
 && tar xzf mpich-3.2.1.tar.gz \
 && rm mpich-3.2.1.tar.gz

run mkdir /pvbuild/build/mpich
workdir /pvbuild/build/mpich

# disable the addition of the RPATH to compiled executables
# this allows us to override the MPI libraries to use those
# found via LD_LIBRARY_PATH

run /pvbuild/src/mpich-3.2.1/configure --prefix=/pvbuild/install/mpich --disable-wrapper-rpath --disable-fortran
run make -j8 
run make install

# add to local environment
env PATH="/pvbuild/install/mpich/bin:$PATH"
env LD_LIBRARY_PATH="/pvbuild/install/mpich/lib:$LD_LIBRARY_PATH"

######################
# ParaView superbuild
######################

run cd /pvbuild/src \
 && git clone --recursive https://gitlab.kitware.com/paraview/paraview-superbuild.git 

workdir /pvbuild/src
run cmake -B /pvbuild/build/paraview -S /pvbuild/src/paraview-superbuild \
  -DCMAKE_BUILD_TYPE_paraview=Release \
  -DCMAKE_INSTALL_PREFIX="/paraview/" \
  -DCMAKE_CXX_COMPILER="/pvbuild/install/gcc/bin/g++" \
  -DCMAKE_C_COMPILER="/pvbuild/install/gcc/bin/gcc" \
  -DENABLE_cxx11=ON \
  -Dparaview_SOURCE_SELECTION:STRING=git \
  -Dparaview_GIT_TAG="v5.9.0" \
  -DENABLE_zfp:BOOL=OFF \
  -DCMAKE_BUILD_TYPE:STRING=Release \
  -DENABLE_netcdf:BOOL=OFF \
  -DENABLE_visitbridge:BOOL=OFF \
  -DENABLE_ffmpeg:BOOL=OFF \
  -DENABLE_qt5:BOOL=OFF \
  -DENABLE_mpi:BOOL=ON \
  -DUSE_SYSTEM_mpi:BOOL=ON \
  -DENABLE_paraview:BOOL=ON \
  -DENABLE_paraviewsdk:BOOL=ON \
  -DENABLE_xdmf3:BOOL=OFF \
  -DBUILD_SHARED_LIBS:BOOL=ON \
  -DENABLE_vtkm:BOOL=OFF \
  -DENABLE_python3:BOOL=ON \
  -DUSE_SYSTEM_python3:BOOL=OFF \
  -DENABLE_pybind11:BOOL=OFF \
  -DENABLE_numpy:BOOL=ON \
  -DUSE_SYSTEM_numpy:BOOL=OFF \
  -DENABLE_scipy:BOOL=OFF \
  -DENABLE_matplotlib:BOOL=ON \
  -DENABLE_vrpn:BOOL=OFF \
  -DENABLE_cosmotools:BOOL=OFF \
  -DENABLE_osmesa:BOOL=ON \
  -Dmesa_USE_SWR:BOOL=ON \
  -DENABLE_tbb:BOOL=ON \
  -DENABLE_silo:BOOL=OFF \
  -DENABLE_boost:BOOL=OFF \
  -DENABLE_vortexfinder2:BOOL=OFF \
  -DENABLE_las:BOOL=OFF \
  -DBUILD_TESTING:BOOL=ON \
  -DENABLE_adios2:BOOL=OFF \
  -DENABLE_ospray:BOOL=ON \
  -DENABLE_ospraymodulempi:BOOL=ON \
  -DENABLE_fontconfig:BOOL=ON \
  -DENABLE_bzip2:BOOL=ON \
  -DPARAVIEW_BUILD_EDITION:STRING=CATALYST_RENDERING \
  -DSUPERBUILD_DEFAULT_INSTALL:STRING=paraviewsdk/TGZ \
  -DPARAVIEW_EXTRA_CMAKE_ARGUMENTS="-DCMAKE_SKIP_INSTALL_RPATH=ON" \
  -DPARAVIEW_EXTRA_CMAKE_ARGUMENTS="-DPARAVIEW_RELOCATABLE_INSTALL=OFF"

workdir /pvbuild/build/paraview

run make download-all 
run make -j8
run make install

#delete build files to save space
#this will also delete local mpich
#and we will need to make sure
#we can dynamically link with native mpi libraries
#run rm -rf /pvbuild

run rm -rf /pvbuild/src
run rm -rf /pvbuild/install/libtool
run rm -rf /pvbuild/install/cmake
run rm -rf /pvbuild/install/gcc/bin /pvbuild/install/gcc/include \
           /pvbuild/install/gcc/lib /pvbuild/install/gcc/libexec \
           /pvbuild/install/gcc/share
run rm -rf /pvbuild/build/gcc /pvbuild/build/mpich /pvbuild/build/libtool
# keep only install directory
run cd /pvbuild/build/paraview ; rm -rf [!i]*
run rm -rf /pvbuild/install/mpich

env LD_LIBRARY_PATH="/pvbuild/install/gcc/lib64"
env PATH="$ORIGINAL_PATH"
env ORIGINAL_PATH=""

workdir /paraview/bin
