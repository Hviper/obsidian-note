
python打包分为可执行包和非执行包，可以把命令行中的执行打包成一个包，这样方便移植，区别是pyproject.toml文件中[project.scripts]配置
![[Pasted image 20260403212550.png]]

## 不可执行包
![[Pasted image 20260403212450.png]]


## 可执行包
![[Pasted image 20260403212456.png]]



![[Pasted image 20260404105959.png]]

  ✅ 正确方法：查看entry_points.txt文件 （可查看python包是可执行包还是不可执行包）                                                         

  pytest（可执行包）：                                                                               
  $ cat /opt/anaconda3/lib/python3.12/site-packages/pytest-7.4.4.dist-info/entry_points.txt          
  [console_scripts]                                                                                  
  py.test = pytest:console_main                                                                      
  pytest = pytest:console_main    # ← 定义了命令 ✅                                                  
  requests（库包）：                                                                                 
  $ cat /opt/anaconda3/lib/python3.12/site-packages/requests-2.32.3.dist-info/entry_points.txt       
  cat: No such file or directory    # ← 文件不存在 ❌   
  
**注意：解压whl包可以看到**