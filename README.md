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
