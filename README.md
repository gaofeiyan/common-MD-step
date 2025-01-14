# MD-step

  ##第一步是引入力场，生成top文件
gmx pdb2gmx -f protein_prepared.pdb -o protein.gro -p topol.top -ignh
  ##选择amber14力场
##输入2
  ##选择tip3的水模型
##输入1
  ##第二步是生成限制文件posre_lig.itp，用来限制蛋白质在整个体系中的运动
gmx genrestr -f molecule.gro -o posre_lig.itp
  ##选择对整个体系进行限制，默认是xyz方向各1000k的力
##选0
  ##将配体itp文件引入限制文件的参数中，针对配体也进行限制
##修改ligand的itp文件，在最末尾添加以下命令  
#ifdef POSRES
#include "posre_lig.itp"
#endif
  ##将配体的top文件与蛋白质top文件进行合并 合并到蛋白质文件
##在topol.top文件的#include "amber14sb_parmbsc1.ff/forcefield.itp"后面空行添加以下命令
; Include ligand topologies
#include "lig1.itp"
##把ligand的名字添加到末尾

  ##将配体的gro文件与蛋白质gro文件进行合并
##将ligand的gro文件中MOL行复制到protein.gro文件倒数第二行，并将protein.gro的原子数相加
  ##生成盒子，并且填充水
gmx editconf -f protein.gro -o complex_box.gro -d 1.0 -bt cubic
gmx solvate -cp complex_box.gro -o complex_sol.gro -p topol.top

  ##生成em.tpr文件用于修改体系电荷
gmx grompp -f em.mdp -c complex_sol.gro -p topol.top -o em.tpr -maxwarn 10
  ##添加离子，使体系平衡
gmx genion -s em.tpr -p topol.top -o system.gro -neutral
  ##选水，意味着将部分水替换成离子
##选15
  ##以下为能量极小化的准备与正式运行
gmx grompp -f em.mdp -c system.gro -p topol.top -o em.tpr 
gmx mdrun -v -deffnm em
  ##以下为限制性动力学的准备与正式运行
gmx grompp -f pr.mdp -c em.gro -p topol.top -r em.gro -o pr.tpr -maxwarn 10
gmx mdrun -v -deffnm pr
  ##以下为分类文件名字的修改与重新定义
gmx make_ndx -f pr.gro
##输入1 | 13
##输入name 22 protein_lig
##输入21
##输入name 23 envir
##输入q
  ##这是正式分子动力学模拟的准备
gmx grompp -f md.mdp -c pr.gro -p topol.top -o md.tpr -n index.ndx -maxwarn 5

##md.mdp文件中要修改：(or)
##comm-grps  = Protein_MOL  tc_grps = Protein_MOL Water_and_ions

  ##进行成品MD模拟(用gpu进行加速）
gmx mdrun -deffnm md -nb gpu -v 


