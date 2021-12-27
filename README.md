# TRAMD
commands for running TRAMD

remove CONNECT lines from ligand-pdb file.

gmx pdb2gmx -f protein.pdb -o protein.gro -ignh
select force filed [Recommended OPLS Force Field]
Select water model [SPC recommended]

antechamber -i ligand.pdb -o ligand.mol2 -fi pdb -fo mol2 -c bcc -nc 0
acpype -di ligand.mol2 -c bcc -n 0

cd ligand.acpype
cp ligand_GMX.gro ligand_GMX.top ligand_GMX.itp ../
cd ../
mv ligand_GMX.gro ligand.gro
mv ligand_GMX.itp ligand.itp
mv ligand_GMX.top ligand.top

cp protein.gro complex.gro
cp topol.top complex.top

cat complex.top | sed '/forcefield\.itp\"/a\
#include "ligand.itp"
' >| complex2.top

echo "Ligand   1" >> complex2.top

mv complex2.top complex.top

#add ligand co-ordiantes to complex.gro 

gmx editconf -f complex.gro -o newbox.gro -bt cubic -d 1
gmx solvate -cp newbox.gro -cs spc216.gro -p complex.top -o solv.gro
gmx grompp -f ions.mdp -c solv.gro -p complex.top -o ions.tpr
gmx genion -s ions.tpr -o solv_ions.gro -p complex.top -pname NA -nname CL -neutral

gmx make_ndx -f solv_ions.gro -o index.ndx 
for protein & ligand. waters & ions

include posere_ions.itp obt from pdb2gmx
include posre_mg.itp in complex.top in contrainst section

# For constriant
gmx genrestr -f solv_ions.gro -n index.ndx -o posre_mg.itp -fc 1000 1000 1000

# Minimization
gmx grompp -f em.mdp -c solv_ions.gro -p complex.top -o em.tpr
gmx mdrun -v -deffnm em -nb gpu
gmx energy -f em.edr -o potential.xvg
select potential (10)
xmgrace potential.xvg

# Equilibration

add below in nvt.mdp file
freezegrps = MG
freezedim  = Y Y Y


gmx grompp -f nvt.mdp -c em.gro -r em.gro -p complex.top -n index.ndx -o nvt.tpr
gmx mdrun -deffnm nvt -nb gpu
