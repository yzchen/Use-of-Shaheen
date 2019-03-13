## Secrets of Shaheen

Some of following rules are applied to [Shaheen-II](https://www.hpc.kaust.edu.sa/content/shaheen-ii) only,
some of them can be used in all [Cray](https://www.cray.com/) HPC systems.

### Shaheen specific

1. Additional login node

    ```
    ssh cdl5
    ```

    `cdl5` is one additional login node, it's not public to users actually, but in some cases, if Shaheen login nodes(`cdl1-4`) are very busy(many people are doing some program compiling or pre-processing staff), go here and you will have a quiet machine.

    **Note : it's not a good idea to occupy this node for so long, it's forbidden to run jobs here!!!**

2. Additional testing rack

    ```
    ssh osprey1
    ssh osprey2
    ssh osprey3

    ssh gateway3
    ```

    Osprey is a 1 rack Cray XC40-AC system. You will find some new cpu bands like knl, kepler, run `sinfo`, the partition name is the cpu band name.

    And usually you have `gateway1-2` as gateway nodes, once you do `salloc`, you will go to one of this two nodes. Gateway3 is for Osprey, so it seems useless for general users.

    **Note : Use of this system is limited to users who have been properly authorised by the School Supercomputing Laboratory. Unauthorised users must disconnect immediately.**

3. Tips on Shaheen (Many of these tips are about linux commands which can be used in all other linux systems)

    `https://www.hpc.kaust.edu.sa/tip`

    You will find many tips here for your easy life.


### Cray applicable

1. File stripe (If you use lustre file system)

    ```
    lfs setstripe --count [stripe-count] filename/directory
    ```

    By default files on Shaheen has stripe `1`, which is good if you don't share the same file for many processes or your file is small enough.

    How to compute stripe count :

        1. if the file is 90GB, the square root is 9.5, so use at least 9

        2. if using 64 MPI processes for parallel I/O, use stripe count less than 64

        3. maximum stripe count on Shaheen-II is 144

2. How to get full allocated node list

    Usually when one requests several nodes, inside the code, `$SLURM_NNODES` and `$SLURM_NODELIST` are set to proper values, for example :

    ```
    cheny0l@cdl2:/project/k1341/caffe/tuning/scripts> srun --time=1:00:00 --nodes=2 -A k1210 --pty bash -l

    cheny0l@nid00191:/project/k1341/caffe/tuning/scripts> echo $SLURM_NNODES
    2

    cheny0l@nid00191:/project/k1341/caffe/tuning/scripts> echo $SLURM_NODELIST
    nid00[191,200]

    ```

    Actually `$SLURM_NODELIST` is in compressed version, which is not good for MPI program.
    This can be a problem when you need to run program without `srun` (apparently this compressed format is designed for slurm system).
    If you need expanded nodelist, try following :

    ```
    cheny0l@nid00191:/project/k1341/caffe/tuning/scripts> scontrol show hostnames $SLURM_NODELIST
    nid00191
    nid00200
    ```

3. Use `mpiexec.hydra` as usual

    Sometimes you don't want to launch your job through `srun`, you want to run it with your own specified openmpi or intel ompi,
    however, with these launch managers, you can **not** run them as following :

    ```
    mpiexec.hydra -host nid00008 -n 1 ./a.out
    ```

    if you do, you will end up with error like `Connection to nid00008 closed by remote host` or `Permission denied`.

    But you do actually want to specify hosts by yourself or by your program, you can run :

    ```
    mpiexec.hydra -bootstrap slurm -host nid00008 -n 1 ./a.out
    ```

    reference : [slurm_intel_mpiexec_hydra](https://slurm.schedmd.com/mpi_guide.html#intel_mpiexec_hydra)

4. MPI program get stuck on Shaheen

    I faces an issue that when I requested large number of compute nodes like 256, my program tends to get stuck during executing without any reasons, logs or erros.
    And this is somehow random in my case, everytime gets stuck at different point.

    If you use `cray-mpich`, you can simply increase the MPI buffer size and number of buffers :

    ```
    export MPICH_GNI_MAX_EAGER_MSG_SIZE=131072
    export MPICH_GNI_NUM_BUFS=128
    ```

    For MPICH_GNI_MAX_EAGER_MSG_SIZE, `131072` is already maximum value you can get;
    for MPICH_GNI_NUM_BUFS, default value is `64`, I didn't find any limit about this term, you should set it based on your memory limits.
