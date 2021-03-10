# 开启和关闭  
```
gpstop [-d master_data_directory] [-B parallel_processes] 
       [-M smart | fast | immediate] [-t timeout_seconds] [-r] [-y] [-a] 
       [-l logfile_directory] [-v | -q]
gpstop -m [-d master_data_directory] [-y] [-l logfile_directory] [-v | -q]
gpstop -u [-d master_data_directory] [-l logfile_directory] [-v | -q]
gpstop --host host_name [-d master_data_directory] [-l logfile_directory]
       [-t timeout_seconds] [-a] [-v | -q]
gpstop --version 
gpstop -? | -h | --help
```
停止所有的节点并重启命令为：
gpstop -r  

在修改过 postgresql.conf 和 pg_hba.conf后不关闭Greenplum的加载命令为：  
gpstop -u  
