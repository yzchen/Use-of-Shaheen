## Programming Guide for Shaheen

Some of following rules are applied to [Shaheen-II](https://www.hpc.kaust.edu.sa/content/shaheen-ii) only,
some can be used in all [Cray](https://www.cray.com/) HPC systems or slurm management systems.

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

    You will find many tips which are very helpful here.

4. Some problems of Shaheen currently

    - module `openmpi2`, `openmpi3` and `openmpi4` are unavaible even it's in the module list, when you run :
        ```
        module load openmpi3
        mpicc
        ```

        you will get `Illegal instruction`, maybe it's a bug in dependent module `xpmem`, I don't know the details

    - module `hdf5` doesn't have proper INCLUDE PATH settings, after loaded `hdf5` module, it will set environment variable `INCLUDE`,
        but actually this is not one of PATHs that can be used by compiler system. So if you want to compile your program with `hdf5`,
        you need to set (`CPATH`) or (`C_INCLUDE_PATH` and `CPP_INCLUDE_PATH`) with the value of `INCLUDE`.

    - some modules set only one of `LIBRARY_PATH` and `LD_LIBRARY_PATH`, so sometimes your compilation will fail with some linking errors,
        again I had an issue with `hdf5`, it only set `LD_LIBRARY_PATH`, after I set `LIBRARY_PATH` manually, I got it right. 
        That means sometimes you need to make sure you have both paths.

5. Disk speed on Shaheen

    a. From login node (`cdl` nodes)

    ```
    time sh -c "dd if=/dev/zero of=/disk/path/test bs=64k count=125000 && sync"
    ```

    | Partition       | IO bandwidth    |
    | :-------------: | :--------------:|
    | home            | 627 MB/s        |
    | project         | 1.0 GB/s        |
    | scratch         | 1.0 GB/s        |

    This benchmark is only for squential read/write, and the result for `home` is not stable maybe the reason of cache.
    Above table tells you if you have scripts to pre-process your data (awk, sed on some files that are not so small), you should use `scratch` or `project` partition,
    `home` is slower than the other two.

    When submiting jobs with slurm system on Shaheen, `/scratch/username` will be home for jobs, not `/home/username` anymore.

    b. From running node (by submitting jobs)

    As said above, running nodes only see `scratch` other than `home`, but for running nodes burst buffer can be seen.

    A simple [benchmark program](https://gist.github.com/yzchen/3d7905f380e9c7f6bff5b2d75f16cdb3) is run here to test disk read preformance, C `read` and `fread` are used.
    Same file is put on different disk partition(`scratch`, `project` and `burst buffer`), run above program 20 times to get average bandwidth value:

    | Partition       | read            | fread           |
    | :-------------: | :--------------:| :--------------:|
    | scratch         | 5865.97 MB/s    | 4287.21 MB/s    |
    | project         | 6493.82 MB/s    | 4470.96 MB/s    |
    | burst buffer    | 3388.2  MB/s    | 3241.89 MB/s    |

    Generally, `project` partition is the fastest among this three, why `burst buffer` is not the fastest even it's SSD(here even data is striped, by setting stripe size, still can obtain whole data on single burst buffer node).
    However, here only squential read is benchmarked, stripped data is better for concurrent read, which is not shown here.

### Cray or Slurm applicable

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
    > srun --time=1:00:00 --nodes=2 -A k1210 --pty bash -l

    > echo $SLURM_NNODES
    2

    > echo $SLURM_NODELIST
    nid00[191,200]

    ```

    Actually `$SLURM_NODELIST` is in compressed version, which is not good for MPI program.
    This can be a problem when you need to run program without `srun` (apparently this compressed format is designed for slurm system).
    If you need expanded nodelist, try following :

    ```
    > scontrol show hostnames $SLURM_NODELIST
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

    It uses the `srun` command rather than the default ssh based method to launch the remote Hydra PM service processes.

    reference : [slurm_intel_mpiexec_hydra](https://slurm.schedmd.com/mpi_guide.html#intel_mpiexec_hydra) and [mpi_environment_reference](https://software.intel.com/en-us/mpi-developer-reference-linux-hydra-environment-variables)

    There is also another way to do this, see [here](https://software.intel.com/en-us/articles/how-to-use-slurm-pmi-with-the-intel-mpi-library-for-linux)

    Basically you should export a few environments:

    ```
    export I_MPI_PMI_LIBRARY=/full/path/to/slurm/libpmi.so
    export I_MPI_FABRICS=shm:dapl
    srun -N <num_nodes> -n <num_processes> ./a.out
    ```

    In this case Intel MPI Library Process Manager is not used and only Slurm utilities control the job (launch, control, terminate), so you will lose some good functionalities of Intel MPI PM.

4. MPI program get stuck on Shaheen(Usually happens when too many nodes used at the same time)

    I faced an issue that when I requested large number of compute nodes like 256, my program tends to get stuck during executing without any reasons, logs or erros.
    And this is somehow random in my case, everytime gets stuck at different point.

    If you use `cray-mpich`, you can simply increase the MPI buffer size and number of buffers :

    ```
    export MPICH_GNI_MAX_EAGER_MSG_SIZE=131072
    export MPICH_GNI_NUM_BUFS=128
    ```

    For MPICH_GNI_MAX_EAGER_MSG_SIZE, `131072` is already maximum value you can get;
    for MPICH_GNI_NUM_BUFS, default value is `64`, I didn't find any limit about this term, you should set it based on your memory limits.

    reference : [MPI Optimization](https://www.hpc.kaust.edu.sa/sites/default/files/files/public/HPCSAUDI17/mpi_optimization.pdf)

    If you are using Intel MPI(Hydra PM), can set environment variables :

    ```
    export I_MPI_SHM_CELL_FWD_SIZE=128K
    export I_MPI_SHM_CELL_BWD_SIZE=128K
    export I_MPI_SHM_CELL_EXT_SIZE=128K
    ```

    In addition, 

    ```
    export I_MPI_SHM_CELL_FWD_NUM=16
    export I_MPI_SHM_CELL_BWD_NUM=16
    export I_MPI_SHM_CELL_EXT_NUM_TOTAL=8K
    ```

5. Burst Buffer (Enhancement of 1: File stripe)

    Burst buffer is supported for fast data access, indeed it's SSD instead of normal disk drive.

    Once you have access to burst buffer, you can create a temporary buffer for your data which only exsits only for one job submission,
    or you can create a persistent buffer, this will exist forever(generally, it's not guaranteed), you can use it in many jobs.

    How to create a persistent buffer and copy(don't move, it's better to have a copy on normal disk) your data to that buffer:

    1. Create a persistent buffer, find a place to keep your burst buffer config file, I use `/scratch/username/.burst`,

        ```
        echo "#BB create_persistent name=testData capacity=500GB access_mode=striped type=scratch" >> /scratch/username/.burst/persist.conf

        salloc -N 0 --bbf=/scratch/username/.burst/persist.conf
        ```

        Once you get allocated, you will have your own persistent buffer

    2. Copy your local data to persistent buffer (I copied a directory, file is also accepted),

        ```
        echo "#DW persistentdw name=testData" >> /scratch/username/.burst/datamove.conf
        echo "#DW stage_in source=/local/data/path/ destination=$DW_PERSISTENT_STRIPED_testData/ type=directory" >> /scratch/username/.burst/datamove.conf

        salloc -N 0 --bbf=/scratch/username/.burst/datamove.conf
        ```

        If your data is large, then you will pend for some time for data copying, once your job is started, that means data will be ready for you.

        Then change your codes or applications to access data from `$DW_PERSISTENT_STRIPED_testData`, this will be faster.

    3. Destroy staged data in burst buffer

        ```
        echo "#BB destroy_persistent name=testData" >> /scratch/username/.burst/destroy.conf
        salloc -N 0 --bbf=/scratch/username/.burst/destroy.conf
        ```

        Note that you can request `0` work nodes for burst buffer configurations.

    4. Use it in following jobs

        add following to your job script, below `#SBATCH` commands

        ```
        #DW persistentdw name=testData
        echo $DW_PERSISTENT_STRIPED_testData
        ```

        make sure you access your data through this environment variable `DW_PERSISTENT_STRIPED_dataName`, because absolute path will change for different runs. Actually you can acess the data through the actual path, it changes for different jobs. On Shaheen, it follows following pattern:

        ```
        /var/opt/cray/dws/mounts/batch/{dataName}_{JOBID}_striped_scratch
        ```

        you can obtain job id through `$SLURM_JOBID` environment variable.
    
    Temporal burst buffer can also be used, it will only exist for current job. Access the data through `$DW_JOB_STRIPED`, or use follong pattern on Shaheen:

    ```
    /var/opt/cray/dws/mounts/batch/{JOBID}_striped_scratch
    ```

    By switching `pool=wlm_pool`(82GB) and `pool=sm_pool`(20.14GB), one can specify pool size for each BB node when in striped case. For example, when requesting a buffer with 40GB, with `wlm_pool` only one BB node is needed, but with `sm_pool` will get 2 nodes.

6. `cc` and `CC` wrapper on Cray

    `cc` is c wrapper, `CC` is cpp wrapper, as you may change your programming environment(3 PEs totally, PrgEnv-cray, PrgEnv-gnu, PrgEnv-intel), flags inside `cc`/`CC` may differ.
    It's good to know what's inside generally.

    Usually with `mpicc`/`mpicxx` we can do `mpicc --show`, with `cc`/`CC` you should use `cc --cray-print-opts=all`

    Following is the output in `PrgEnv-intel`:

    ```
    cc --cray-print-opts=all

    -I/opt/cray/pe/libsci/17.12.1/INTEL/16.0/x86_64/include -I/opt/cray/pe/mpt/7.7.0/gni/mpich-intel/16.0/include -I/opt/cray/rca/2.2.18-6.0.7.0_33.3__g2aa4f39.ari/include -I/opt/cray/alps/6.6.43-6.0.7.0_26.4__ga796da3.ari/include -I/opt/cray/xpmem/2.2.15-6.0.7.1_5.11__g7549d06.ari/include -I/opt/cray/gni-headers/5.0.12.0-6.0.7.0_24.1__g3b1768f.ari/include -I/opt/cray/pe/pmi/5.0.13/include -I/opt/cray/ugni/6.0.14.0-6.0.7.0_23.1__gea11d3d.ari/include -I/opt/cray/udreg/2.3.2-6.0.7.0_33.18__g5196236.ari/include -I/opt/cray/wlm_detect/1.3.3-6.0.7.0_47.2__g7109084.ari/include -I/opt/cray/krca/2.2.4-6.0.7.1_5.42__g8505b97.ari/include -I/opt/cray-hss-devel/8.0.0/include -L/opt/cray/pe/libsci/17.12.1/INTEL/16.0/x86_64/lib -L/opt/cray/dmapp/default/lib64 -L/opt/cray/pe/mpt/7.7.0/gni/mpich-intel/16.0/lib -L/opt/cray/rca/2.2.18-6.0.7.0_33.3__g2aa4f39.ari/lib64 -L/opt/cray/pe/atp/2.1.1/libApp -L/lib64 -Wl,--no-as-needed,-lAtpSigHandler,-lAtpSigHCommData -Wl,--undefined=_ATP_Data_Globals -Wl,--undefined=__atpHandlerInstall -lrca -lz -Wl,--as-needed,-lsci_intel_mpi,--no-as-needed -Wl,--as-needed,-lsci_intel,--no-as-needed -Wl,--as-needed,-lmpich_intel,--no-as-needed

    CC --cray-print-opts=all

    -I/opt/cray/pe/libsci/17.12.1/INTEL/16.0/x86_64/include -I/opt/cray/pe/mpt/7.7.0/gni/mpich-intel/16.0/include -I/opt/cray/rca/2.2.18-6.0.7.0_33.3__g2aa4f39.ari/include -I/opt/cray/alps/6.6.43-6.0.7.0_26.4__ga796da3.ari/include -I/opt/cray/xpmem/2.2.15-6.0.7.1_5.11__g7549d06.ari/include -I/opt/cray/gni-headers/5.0.12.0-6.0.7.0_24.1__g3b1768f.ari/include -I/opt/cray/pe/pmi/5.0.13/include -I/opt/cray/ugni/6.0.14.0-6.0.7.0_23.1__gea11d3d.ari/include -I/opt/cray/udreg/2.3.2-6.0.7.0_33.18__g5196236.ari/include -I/opt/cray/wlm_detect/1.3.3-6.0.7.0_47.2__g7109084.ari/include -I/opt/cray/krca/2.2.4-6.0.7.1_5.42__g8505b97.ari/include -I/opt/cray-hss-devel/8.0.0/include -L/opt/cray/pe/libsci/17.12.1/INTEL/16.0/x86_64/lib -L/opt/cray/dmapp/default/lib64 -L/opt/cray/pe/mpt/7.7.0/gni/mpich-intel/16.0/lib -L/opt/cray/dmapp/default/lib64 -L/opt/cray/pe/mpt/7.7.0/gni/mpich-intel/16.0/lib -L/opt/cray/rca/2.2.18-6.0.7.0_33.3__g2aa4f39.ari/lib64 -L/opt/cray/pe/atp/2.1.1/libApp -L/lib64 -Wl,--no-as-needed,-lAtpSigHandler,-lAtpSigHCommData -Wl,--undefined=_ATP_Data_Globals -Wl,--undefined=__atpHandlerInstall -lrca -lz -Wl,--as-needed,-lsci_intel_mpi,--no-as-needed -Wl,--as-needed,-lsci_intel,--no-as-needed -Wl,--as-needed,-lmpich_intel,--no-as-needed -Wl,--as-needed,-lmpichcxx_intel,--no-as-needed
    ```

7. Check out who is the most annoying guy on Shaheen

    Sometimes you submit a small job but it's pending for very long time, in that case it's very annoying.
    So you want to know who occupied most compute nodes on Shaheen.

    ```
    squeue | less
    ```

    Through above command you can see status of all submitted jobs, some are running, some are pending.
    You noticed one guy who has one job array contains one thousand jobs, each needs to run for 24 hours.
    You have username of that guy but no other information, but username on Shaheen is not enough to find a person.

    ```
    finger username
    ```

    However, `finger` command can give actual full name of given user, so you can send an email to him to complain about his jobs :)

8. Make your own modules, make your own environments

    Usually when you need to do some programming staff, you need specific compilers, headers, libbraries and so on. 
    You can execute several `module load xxx` to load all modules, but another solution is to build a specific module that will do all these things for you.

    a. Find a good place, like `/scratch/username/.usr/local/share/modulefiles`, create a file,

    ```
    cd /scratch/username/.usr/local/share/modulefiles
    touch common
 
    echo "#%Module" >> common
    echo "" >> common
    echo "module swap PrgEnv-cray/6.0.4 PrgEnv-intel" >> common
    echo "module load cmake/3.10.2" >> common
    echo "module load hdf5/1.8.21" >> common
   ```

   The first line in file `common` will indicate this is a module file. 
   You can put all `module load` commands in `common` file to let it load all dependencies you want.

    b. In order to use this new module, run following commands:

    ```
    module use -a /scratch/username/.usr/local/share/modulefiles
    module load common
    ```

    Now all modules will be loaded.

    c. Do other settings, not only just loading more modules, new environment variables and some paths can be set in this modules.
    We can append following `common` file or create a new module file just to have a better structure.

    ```
    cd /scratch/cheny0l/.usr/local/share/modulefiles
    mkdir depends
    cd depends
    # file name will denote the version number
    touch 1.0
 
    echo "#%Module" >> 1.0
    echo "" >> 1.0
 
    echo "set     root            /scratch/username/.usr/local" >> 1.0
    echo "prepend-path    PATH            $root/bin" >> 1.0
    echo "prepend-path    CPLUS_INCLUDE_PATH  $root/include" >> 1.0
    echo "prepend-path    C_INCLUDE_PATH      $root/include" >> 1.0
    echo "prepend-path    LD_LIBRARY_PATH     $root/lib:$root/lib64" >> 1.0
    echo "prepend-path    LIBRARY_PATH        $root/lib:$root/lib64" >> 1.0
    echo "prepend-path    MANPATH         $root/share" >> 1.0
 
    cd ../
    # append above file to `common` module
    echo "module load depends/1.0" >> 1.0
    ```

    Now when commands in `b` are executed, not only new modules will be loaded, but also you can have all above paths.

9. Profiling tools available on Cray system

    Cray does provide a way to do program profiling, here I forget I/O profiling(`darshan`). Imagine you have a program and want to do profiling,

    a. load proper modules, `perftools-lite` is a lite module for doing this, you don't need to do anything else with this module, just compile as normal.
    but it has conflict with `darshan` module, so make sure you have removed it before using `perftools-lite`

    ```
    module unload darshan
    module load perftools-lite
    ```

    b. compile program, remember to compile program with Cray compiler wrapper always

    ```
    CC examples.cc -o examples
    ```

    c. submit a job, once job is finished, you can find profiling output in output log file, and some other files for visulization

    Of course there is also full module called `perftools`,

    a. load modules as the same

    ```
    module unload darshan
    module load perftools
    ```

    b. compile program, keep all `*.o` files

    ```
    CC -c examples.o examples.cc
    CC -o exmaples examples.o
    ```
    
    c. run one more build step, one more executable file `examples+pat` will be generated

    ```
    pat_build examples
    ```

    d. submit job with `examples+pat`, use `pat_report` to parse the output report, also some visulizations can be applied here

    Detailed [exmaples and explanations](https://www.nersc.gov/users/software/performance-and-debugging-tools/craypat/)
