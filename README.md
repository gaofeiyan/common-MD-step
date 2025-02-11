** MD-step**

  第一步是引入力场，生成top文件
gmx pdb2gmx -f protein_prepared.pdb -o protein.gro -p topol.top -ignh
  选择amber14力场
输入2
  选择tip3的水模型
输入1
  第二步是生成限制文件posre_lig.itp，用来限制蛋白质在整个体系中的运动
gmx genrestr -f molecule.gro -o posre_lig.itp
  选择对整个体系进行限制，默认是xyz方向各1000k的力
选0
  将配体itp文件引入限制文件的参数中，针对配体也进行限制
修改ligand的itp文件，在最末尾添加以下命令  
ifdef POSRES
include "posre_lig.itp"
endif
  将配体的top文件与蛋白质top文件进行合并 合并到蛋白质文件
在topol.top文件的include "amber14sb_parmbsc1.ff/forcefield.itp"后面空行添加以下命令
; Include ligand topologies
include "lig1.itp"
把ligand的名字添加到末尾

  将配体的gro文件与蛋白质gro文件进行合并
将ligand的gro文件中MOL行复制到protein.gro文件倒数第二行，并将protein.gro的原子数相加
  生成盒子，并且填充水
gmx editconf -f protein.gro -o complex_box.gro -d 1.0 -bt cubic
gmx solvate -cp complex_box.gro -o complex_sol.gro -p topol.top

  生成em.tpr文件用于修改体系电荷
gmx grompp -f em.mdp -c complex_sol.gro -p topol.top -o em.tpr -maxwarn 10
  添加离子，使体系平衡
gmx genion -s em.tpr -p topol.top -o system.gro -neutral   
  选水，意味着将部分水替换成离子
选15
  以下为能量极小化的准备与正式运行
gmx grompp -f em.mdp -c system.gro -p topol.top -o em.tpr 
gmx mdrun -v -deffnm em
  以下为限制性动力学的准备与正式运行
gmx grompp -f pr.mdp -c em.gro -p topol.top -r em.gro -o pr.tpr -maxwarn 10
gmx mdrun -v -deffnm pr
  以下为分类文件名字的修改与重新定义
gmx make_ndx -f pr.gro
输入1 | 13
输入name 22 protein_lig
输入21
输入name 23 envir
输入q
  这是正式分子动力学模拟的准备
gmx grompp -f md.mdp -c pr.gro -p topol.top -o md.tpr -n index.ndx -maxwarn 5

md.mdp文件中要修改：(or)
comm-grps  = Protein_MOL  tc_grps = Protein_MOL Water_and_ions

  进行成品MD模拟(用gpu进行加速）
gmx mdrun -deffnm md -nb gpu -v 


** Result Analyze**

gmx make_ndx -f md.gro -o prolig_center.ndx
首先将蛋白质和配体组合为一组，保存为 prolig_center.ndx

gmx trjconv -s md.tpr -f md.xtc -o prolig_fit.xtc -pbc mol -center -n prolig_center.ndx
运行命令之后先选择校正中心为Protein_MOL，然后选择对整个体系进行校正。输入0

#轨迹可视化
gmx trjconv -s md.tpr -f prolig_fit.xtc -o traj.pdb -n prolig_center.ndx -dt 100 

gmx rms -f prolig_fit.xtc -s md.tpr -o md-rmsd.xvg 

gmx gyrate -s md.tpr -f prolig_fit.xtc -o md-gyrate.xvg 

gmx sasa -f prolig_fit.xtc -s md.tpr -o md-area.xvg 

gmx trjconv -f prolig_fit.xtc -b 60000 -e 100000 -o analyze.xtc 

gmx rmsf -f analyze.xtc -s md.tpr -o rmsf.xvg -res -n prolig_center.ndx

select Backbone group.


gmx hbond -f analyze.xtc -s md.tpr -n prolig_center.ndx -num -hbn -hbm


gmx rms -s md.tpr -f prolig_fit.xtc -o FEL_rmsd.xvg -n prolig_center.ndx
  4 4 
  
gmx gyrate -s md.tpr -f prolig_fit.xtc -o FEL_gyrate.xvg -n prolig_center.ndx
  4
  合并
gmx sham -tsham 310 -nlevels 100 -f output.xvg -ls gibbs.xpm -g gibbs.log -lsh enthalpy.xpm -lss entropy.xpm

gmx trjconv -s md.tpr -f prolig_fit.xtc -o 4194.pdb -sep -b 4100 -e 4100 -pbc mol -n prolig_center.ndx

python xpm2png.py -ip yes -f gibbs.xpm (sources/xpm_show/xpm2png.py)

 MM/PBSA
chmod +x gmx_mmpbsa.bash

bash gmx_mmpbsa.bash





