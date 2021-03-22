# seeing_threads

```
#include <iostream>
#include <omp.h>

int main(int argc, char* argv[]) {
  using namespace std;
 
  #pragma omp parallel
  {
  int id = omp_get_thread_num();
  int nthrds = omp_get_num_threads();
  cout << "Hello from thread " << id << " of " << nthrds << endl;
  while (true) {};
  }
  return 0;
}
```

```
#!/bin/bash
#SBATCH --job-name=cxx_omp       # create a short name for your job
#SBATCH --nodes=1                # node count
#SBATCH --ntasks=1               # total number of tasks across all nodes
#SBATCH --cpus-per-task=4        # cpu-cores per task (>1 if multi-threaded tasks)
#SBATCH --mem-per-cpu=4G         # memory per cpu-core (4G per CPU-core is default)
#SBATCH --time=00:00:10          # total run time limit (HH:MM:SS)

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

module purge
module load intel/19.1/64/19.1.1.217

./hw_omp
```



```
$ ssh <compute-node>
$ htop -u jdh4
  PID USER  PRI  NI  VIRT   RES   SHR S CPU% MEM%   TIME+  Command
26464 jdh4   21   1  110M  1500  1228 S  0.0  0.0  0:00.00 │  ├─ /bin/bash /var/spool/slurmd/job33903377/slurm_script
26469 jdh4	 21   1 23204  1344  1116 R 398.  0.0  5:50.75 │  │  └─ ./hw_omp
26472 jdh4	 21   1 23204  1344  1116 R 99.7  0.0  1:27.77 │  │     ├─ ./hw_omp
26471 jdh4	 21   1 23204  1344  1116 R 98.4  0.0  1:27.69 │  │     ├─ ./hw_omp
26470 jdh4	 21   1 23204  1344  1116 R 99.7  0.0  1:27.78 │  │     └─ ./hw_omp
```

We see that `htop` seems to show 3 threads instead of 4. It appears that the fourth is tied to the overall process (26469).

```
$ ps -eLf | head -n 1
UID        PID  PPID   LWP  C NLWP STIME TTY          TIME CMD
[jdh4@della-r4c4n3 ~]$ ps -eLf | grep jdh4
jdh4     26464 26459 26464  0    1 15:22 ?        00:00:00 /bin/bash /var/spool/slurmd/job33903377/slurm_script
jdh4     26469 26464 26469 99    4 15:22 ?        00:05:14 ./hw_omp
jdh4     26469 26464 26470 99    4 15:22 ?        00:05:15 ./hw_omp
jdh4     26469 26464 26471 99    4 15:22 ?        00:05:15 ./hw_omp
jdh4     26469 26464 26472 99    4 15:22 ?        00:05:15 ./hw_omp
root     26549  3141 26549  0    1 15:23 ?        00:00:00 sshd: jdh4 [priv]
jdh4     26564 26549 26564  0    1 15:23 ?        00:00:00 sshd: jdh4@pts/0
jdh4     26565 26564 26565  0    1 15:23 pts/0    00:00:00 -bash
jdh4     27125 26565 27125  0    1 15:27 pts/0    00:00:00 ps -eLf
jdh4     27126 26565 27126  0    1 15:27 pts/0    00:00:00 grep --color=auto jdh4
```



```
$ top -H -u jdh4
  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND 
26469 jdh4      21   1   23204   1344   1116 R 99.9  0.0   6:04.96 hw_omp                                                                           
26471 jdh4      21   1   23204   1344   1116 R 99.9  0.0   6:06.17 hw_omp                                                                           
26472 jdh4      21   1   23204   1344   1116 R 99.9  0.0   6:06.32 hw_omp                                                                           
26470 jdh4      21   1   23204   1344   1116 R 99.3  0.0   6:06.11 hw_omp                                                                           
27160 jdh4      20   0  173988   3552   1684 R  0.3  0.0   0:00.18 top                                                                             
26464 jdh4      21   1  113288   1500   1228 S  0.0  0.0   0:00.00 slurm_script                                                                     
26564 jdh4      20   0  175912   2792   1112 S  0.0  0.0   0:00.01 sshd                                                                             
26565 jdh4      20   0  127028   2860   1684 S  0.0  0.0   0:00.02 bash 
```

## Hybrid OpenMP/MPI

```
#include <iostream>
#include <mpi.h>
#include <omp.h>

int main(int argc, char** argv) {
  using namespace std;
  
  MPI_Init(&argc, &argv);

  int world_size, world_rank;
  MPI_Comm_size(MPI_COMM_WORLD, &world_size);
  MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);

  // Get the name of the processor
  char processor_name[MPI_MAX_PROCESSOR_NAME];
  int name_len;
  MPI_Get_processor_name(processor_name, &name_len);

  #pragma omp parallel
  {
  int id = omp_get_thread_num();
  int nthrds = omp_get_num_threads();
  cout << "Hello from thread " << id << " of " << nthrds
       << " on MPI process " << world_rank << " of " << world_size
       << " on node " << processor_name << endl;
       while (true) {};
  }

  MPI_Finalize();
  return 0;
}
```

```
#!/bin/bash
#SBATCH --job-name=cxx_mpi_omp   # create a short name for your job
#SBATCH --nodes=1                # node count
#SBATCH --ntasks-per-node=2      # number of tasks per node
#SBATCH --cpus-per-task=3        # cpu-cores per task (>1 if multi-threaded tasks)
#SBATCH --mem-per-cpu=4G         # memory per cpu-core (4G is default)
#SBATCH --time=00:15:00          # total run time limit (HH:MM:SS)

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

module purge
module load intel/19.1/64/19.1.1.217 intel-mpi/intel/2019.7/64

srun ./mpi_openmp_hello_world
```

The below shows that there a two processes with two explicit threads per process and the third is combined into the overall two processes:

```
  PID USER      PRI  NI  VIRT   RES   SHR S CPU% MEM%   TIME+  Command
14245 jdh4       21   1 2479M 42884  5844 R 299.  0.0  9:29.93 │  ├─ /home/jdh4/software/hpc_beginning_workshop/RC_example_jobs/cxx/hybrid_multithreaded_parallel/./mpi_openmp_hello_world
14273 jdh4	 21   1 2479M 42884  5844 R 99.3  0.0  3:09.36 │  │  ├─ /home/jdh4/software/hpc_beginning_workshop/RC_example_jobs/cxx/hybrid_multithreaded_parallel/./mpi_openmp_hello_world
14272 jdh4       21   1 2479M 42884  5844 R 100.  0.0  3:10.08 │  │  └─ /home/jdh4/software/hpc_beginning_workshop/RC_example_jobs/cxx/hybrid_multithreaded_parallel/./mpi_openmp_hello_world
14244 jdh4	 21   1 2477M 44944  5852 R 299.  0.0  9:31.31 │  ├─ /home/jdh4/software/hpc_beginning_workshop/RC_example_jobs/cxx/hybrid_multithreaded_parallel/./mpi_openmp_hello_world
14275 jdh4	 21   1 2477M 44944  5852 R 100.  0.0  3:10.42 │  │  ├─ /home/jdh4/software/hpc_beginning_workshop/RC_example_jobs/cxx/hybrid_multithreaded_parallel/./mpi_openmp_hello_world
14274 jdh4	 21   1 2477M 44944  5852 R 100.  0.0  3:10.42 │  │  └─ /home/jdh4/software/hpc_beginning_workshop/RC_example_jobs/cxx/hybrid_multithreaded_parallel/./mpi_openmp_hello_world
14223 jdh4	 21   1  110M  1504  1228 S  0.0  0.0  0:00.00 │  ├─ /bin/bash /var/spool/slurmd/job33903909/slurm_script
14228 jdh4	 21   1  322M  5548  2404 S  0.0  0.0  0:00.00 │  │  └─ srun ./mpi_openmp_hello_world
14233 jdh4	 21   1  322M  5548  2404 S  0.0  0.0  0:00.00 │  │     ├─ srun ./mpi_openmp_hello_world
14232 jdh4	 21   1  322M  5548  2404 S  0.0  0.0  0:00.00 │  │     ├─ srun ./mpi_openmp_hello_world
14231 jdh4	 21   1  322M  5548  2404 S  0.0  0.0  0:00.00 │  │     ├─ srun ./mpi_openmp_hello_world
14230 jdh4	 21   1  322M  5548  2404 S  0.0  0.0  0:00.00 │  │     ├─ srun ./mpi_openmp_hello_world
14229 jdh4	 21   1 47928   784     0 S  0.0  0.0  0:00.00 │  │     └─ srun ./mpi_openmp_hello_world
14324 jdh4	 20   0  171M  2788  1108 S  0.0  0.0  0:00.00 │     └─ sshd: jdh4@pts/0
14325 jdh4	 20   0  124M  2956  1720 S  0.0  0.0  0:00.02 │        └─ -bash
14626 jdh4       20   0  131M  3520  1556 R  0.7  0.0  0:00.23 │           └─ htop -u jdh4 -u jdh4
```

The below shows

```
$ ps -eLf | head -n 1
UID        PID  PPID   LWP  C NLWP STIME TTY          TIME CMD
[jdh4@della-r1c3n8 hybrid_multithreaded_parallel]$ ps -eLf | grep jdh4
jdh4     14223 14218 14223  0    1 15:50 ?        00:00:00 /bin/bash /var/spool/slurmd/job33903909/slurm_script
jdh4     14228 14223 14228  0    5 15:50 ?        00:00:00 srun ./mpi_openmp_hello_world
jdh4     14228 14223 14230  0    5 15:50 ?        00:00:00 srun ./mpi_openmp_hello_world
jdh4     14228 14223 14231  0    5 15:50 ?        00:00:00 srun ./mpi_openmp_hello_world
jdh4     14228 14223 14232  0    5 15:50 ?        00:00:00 srun ./mpi_openmp_hello_world
jdh4     14228 14223 14233  0    5 15:50 ?        00:00:00 srun ./mpi_openmp_hello_world
jdh4     14229 14228 14229  0    1 15:50 ?        00:00:00 srun ./mpi_openmp_hello_world
jdh4     14244 14237 14244 99    3 15:50 ?        00:04:52 /home/jdh4/software/hpc_beginning_workshop/RC_example_jobs/cxx/hybrid_multithreaded_parallel/./mpi_openmp_hello_world
jdh4     14244 14237 14274 99    3 15:50 ?        00:04:52 /home/jdh4/software/hpc_beginning_workshop/RC_example_jobs/cxx/hybrid_multithreaded_parallel/./mpi_openmp_hello_world
jdh4     14244 14237 14275 99    3 15:50 ?        00:04:52 /home/jdh4/software/hpc_beginning_workshop/RC_example_jobs/cxx/hybrid_multithreaded_parallel/./mpi_openmp_hello_world
jdh4     14245 14237 14245 99    3 15:50 ?        00:04:52 /home/jdh4/software/hpc_beginning_workshop/RC_example_jobs/cxx/hybrid_multithreaded_parallel/./mpi_openmp_hello_world
jdh4     14245 14237 14272 99    3 15:50 ?        00:04:51 /home/jdh4/software/hpc_beginning_workshop/RC_example_jobs/cxx/hybrid_multithreaded_parallel/./mpi_openmp_hello_world
jdh4     14245 14237 14273 99    3 15:50 ?        00:04:50 /home/jdh4/software/hpc_beginning_workshop/RC_example_jobs/cxx/hybrid_multithreaded_parallel/./mpi_openmp_hello_world
root     14311  3210 14311  0    1 15:51 ?        00:00:00 sshd: jdh4 [priv]
jdh4     14324 14311 14324  0    1 15:51 ?        00:00:00 sshd: jdh4@pts/0
jdh4     14325 14324 14325  0    1 15:51 pts/0    00:00:00 -bash
jdh4     14793 14325 14793  0    1 15:55 pts/0    00:00:00 ps -eLf
jdh4     14794 14325 14794  0    1 15:55 pts/0    00:00:00 grep --color=auto jdh4
```

```
$ top -u jdh4
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                                                     
14244 jdh4      21   1 2537436  44944   5852 R 300.0  0.0  25:24.54 mpi_openmp_hell                                                                                                                             
14245 jdh4      21   1 2539480  42884   5844 R 299.7  0.0  25:21.98 mpi_openmp_hell                                                                                                                             
15647 jdh4      20   0  173172   2748   1672 R   0.7  0.0   0:00.03 top                                                                                                                                         
14223 jdh4      21   1  113428   1504   1228 S   0.0  0.0   0:00.00 slurm_script                                                                                                                                
14228 jdh4      21   1  330080   5548   2404 S   0.0  0.0   0:00.00 srun                                                                                                                                        
14229 jdh4      21   1   47928    784      0 S   0.0  0.0   0:00.00 srun                                                                                                                                        
14324 jdh4      20   0  175912   2788   1108 S   0.0  0.0   0:00.02 sshd                                                                                                                                        
14325 jdh4      20   0  127032   2956   1720 S   0.0  0.0   0:00.03 bash  
```

```
$ top -u jdh4 -H
  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                                                                                                                                      
14244 jdh4      21   1 2537436  44944   5852 R 99.9  0.0   9:00.84 mpi_openmp_hell                                                                                                                              
14274 jdh4      21   1 2537436  44944   5852 R 99.9  0.0   9:00.82 mpi_openmp_hell                                                                                                                              
14275 jdh4      21   1 2537436  44944   5852 R 99.9  0.0   9:00.83 mpi_openmp_hell                                                                                                                              
14245 jdh4      21   1 2539480  42884   5844 R 99.9  0.0   9:00.88 mpi_openmp_hell                                                                                                                              
14273 jdh4      21   1 2539480  42884   5844 R 99.9  0.0   8:59.20 mpi_openmp_hell                                                                                                                              
14272 jdh4      21   1 2539480  42884   5844 R 88.2  0.0   8:59.81 mpi_openmp_hell                                                                                                                              
15697 jdh4      20   0  174040   3488   1624 R 17.6  0.0   0:00.03 top                                                                                                                                          
14223 jdh4      21   1  113428   1504   1228 S  0.0  0.0   0:00.00 slurm_script                                                                                                                                 
14228 jdh4      21   1  330080   5548   2404 S  0.0  0.0   0:00.00 srun                                                                                                                                         
14230 jdh4      21   1  330080   5548   2404 S  0.0  0.0   0:00.00 srun                                                                                                                                         
14231 jdh4      21   1  330080   5548   2404 S  0.0  0.0   0:00.00 srun                                                                                                                                         
14232 jdh4      21   1  330080   5548   2404 S  0.0  0.0   0:00.00 srun                                                                                                                                         
14233 jdh4      21   1  330080   5548   2404 S  0.0  0.0   0:00.00 srun                                                                                                                                         
14229 jdh4      21   1   47928    784      0 S  0.0  0.0   0:00.00 srun                                                                                                                                         
14324 jdh4      20   0  175912   2788   1108 S  0.0  0.0   0:00.02 sshd                                                                                                                                         
14325 jdh4      20   0  127032   2956   1720 S  0.0  0.0   0:00.03 bash
```
