Prerequisite

```bash
module load cmake
module load gcc/5.4.0
module load cuda
```

Install Kokkos
```bash
KOKKOS_INSTALL_DIR=/ccs/home/againaru/kokkos/install_new
OPTIONS=(
    -D CMAKE_INSTALL_PREFIX="${KOKKOS_INSTALL_DIR}"
    -D Kokkos_ENABLE_SERIAL=ON
    -D Kokkos_ENABLE_OPENMP=ON
    -D Kokkos_ENABLE_CUDA=ON
        -D Kokkos_ENABLE_CUDA_LAMBDA=ON
    -D Kokkos_ARCH_POWER9=ON
    -D Kokkos_ARCH_VOLTA70=ON
    )
cmake "${OPTIONS[@]}" ../kokkos
make -j4
make intall -j4
```

Install ArborX
```bash
cmake -D CMAKE_INSTALL_PREFIX=/ccs/home/againaru/arborx/ArborX/install -D ARBORX_ENABLE_MPI=ON -D CMAKE_PREFIX_PATH=/ccs/home/againaru/kokkos/install_new -D CMAKE_CXX_COMPILER=/ccs/home/againaru/kokkos/install_new/bin/nvcc_wrapper -D CMAKE_CXX_EXTENSIONS=OFF ..

To run the exampes an additional parameter is required
-D ARBORX_ENABLE_EXAMPLES=ON

make -j4
```
