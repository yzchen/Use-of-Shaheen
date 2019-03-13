## Secrets of Shaheen

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

3. Tips on Shaheen

    `https://www.hpc.kaust.edu.sa/tip`

    You will find many tips here for your easy life.

4. File stripe

    ```
    lfs setstripe --count [stripe-count] filename/directory
    ```

    By default files on Shaheen has stripe `1`, which is good if you don't share the same file for many processes or your file is small enough.

    How to compute stripe count : 
    
        1. if the file is 90GB, the square root is 9.5, so use at least 9

        2. if using 64 MPI processes for parallel I/O, use stripe count less than 64

        3. maximum stripe count on Shaheen-II is 144

5. How to get full allocated node list

    Usually when one requests several nodes, inside the code, `$SLURM_NNODES` and `$SLURM_NODELIST` are set to proper values, for example :

    ```
    cheny0l@cdl2:/project/k1341/caffe/tuning/scripts> srun --time=1:00:00 --nodes=2 -A k1210 --pty bash -l 

    cheny0l@nid00191:/project/k1341/caffe/tuning/scripts> echo $SLURM_NNODES 
    2

    cheny0l@nid00191:/project/k1341/caffe/tuning/scripts> echo $SLURM_NODELIST 
    nid00[191,200]

    ```

    Actually `$SLURM_NODELIST` is in compressed version, which is not good for MPI program.
    This can be a problem when you need to run program without `srun` because apparently `srun` can resolve this issue.
    If eventually you need expanded nodelist, try following :

    ```
    cheny0l@nid00191:/project/k1341/caffe/tuning/scripts> scontrol show hostnames $SLURM_NODELIST
    nid00191
    nid00200
    ```
