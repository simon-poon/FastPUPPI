Basic Instructions

```
cmsrel CMSSW_12_3_0_pre4
cd CMSSW_12_3_0_pre4/src
cmsenv
git cms-checkout-topic -u p2l1pfp:L1PF_12_3_X
git cms-addpkg L1Trigger/TrackTrigger
git cms-addpkg SimTracker/TrackTriggerAssociation
git cms-addpkg L1Trigger/L1TMuon
mv L1Trigger/L1TMuon/data L1Trigger/L1TMuon/data.old
git clone https://github.com/cms-l1t-offline/L1Trigger-L1TMuon.git L1Trigger/L1TMuon/data

# scripts
git clone git@github.com:p2l1pfp/FastPUPPI.git -b 12_3_X

scram b -j8
```

If you start from GEN-SIM-DIGI-RAW, the first step is to produce the inputs files containing the basic TPs:
```
cd FastPUPPI/NtupleProducer/python/
cmsRun runInputs110X.py 
```
Currently only  `11_0_X` from the HLT TDR campaign (Phase2C9, Geometry D49, HGCal v11) are supported.
 * The old input files from processing `11_0_X` HLT TDR samples in `CMSSW_11_1_6`, referred to as `110X_v2` can be used from this `12_3_X` branch and gives the same output as from the old `11_1_X` branch.
 * A new set of input files from processing `11_0_X` HLT TDR samples in this `CMSSW_12_3_X` branch is in preparation, to benefit from the TP improvements

The second step runs the algorithms on the input files and creates ntuples which can be used to do analysis.
All python configuration files for CMSSW are under `NtupleProducer/python`, while standalone python scripts or fwlite macros are under `NtupleProducer/python/scripts`. 
In order to run the python configuration file on many events locally, a driver script `scripts/prun.sh` can be used to run locally the python configuration files, which takes care of selecting the input files, splitting the task to run on multiple CPUs and merge the result.

1) Ntuple for trigger rate studies:

```
cmsRun runRespNTupler.py
```

To run the ntuplizer over many files, from within `NtupleProducer/python` do for instance:
```
./scripts/prun.sh runPerformanceNTuple.py --110X_v2 TTbar_PU200 TTbar_PU200.110X_v2  --inline-customize 'oldInputs()'
./scripts/prun.sh runPerformanceNTuple.py --110X_v2 TTbar_PU0 TTbar_PU0.110X_v2  --inline-customize 'oldInputs();goGun()'
./scripts/prun.sh runPerformanceNTuple.py --110X_v2 MultiPion_PT0to200_PU0 MultiPion_PT0to200_PU0.110X_v2  --inline-customize 'oldInputs()goGun()'
./scripts/prun.sh runPerformanceNTuple.py --110X_v2 MultiPion_PT0to120_gun_PU200 MultiPion_PT0to120_gun_PU200.110X_v2  --inline-customize 'oldInputs();goGun();noPU()'
```
Look into the prun.sh script to check the paths to the input files and the corresponding options.

NB: 
   * When processing samples where TPs were produced in `11_1_6`, add `--inline-customize 'oldInputs()'`
   * For samples without pileup, add  `--inline-customize 'noPU()'` to the prun.sh command line or add `noPU()` at the end of the file
   * For single particle samples `goGun()` (and if at PU0 also `noPU()`)


The third step is to produce the plots from the ntuple. The plotting scripts are in:
```FastPUPPI/NtupleProducer/python/scripts```

1) For single particle or jet response:

```
python scripts/respPlots.py respTupleNew_SinglePion_PU0.110X_v2.root plots_dir -w l1pfw -p pion
python scripts/respPlots.py respTupleNew_TTbar_PU200.110X_v2.root plots_dir -w l1pf -p jet
```

2) For jet, MET and HT plots, the first step is to produce JECs
```
python scripts/makeJecs.py perfNano_TTbar_PU200.root -A -o jecs.root
```
then you can use `jetHtSuite.py` to make the plots

```
python scripts/jetHtSuite.py perfNano_TTbar_PU200.root perfNano_SingleNeutrino_PU200.root plots_dir -w l1pfpu -v ht
python scripts/jetHtSuite.py perfNano_TTbar_PU200.root perfNano_SingleNeutrino_PU200.root plots_dir -w l1pfpu -v met
python scripts/jetHtSuite.py perfNano_TTbar_PU200.root perfNano_SingleNeutrino_PU200.root plots_dir -w l1pfpu -v jet4
```

3) For object multiplicitly studies (this may need to be updated):

Plot the number of elements in all subdetectors: 
```
python scripts/objMultiplicityPlot.py perfTuple_TTbar_PU200.root  plotdir -s TTbar_PU200  
```
Notes:
 * you can select only some subdetector with `-d`, or some kind of object with `-c`.

4) For lepton efficiency studies (preliminary)

* produce the perfNano file switching on gen and reco leptons, e.g. adding `addGenLep();addPFLep([13],["PF"]);addStaMu();addTkEG()`.
* `jetHtSuite.py` can be used to make lepton efficiency and rate plots:
  * `-P lepeff -v lep_V --xvar lep_X` can make a plot of efficiency vs generated X (e.g. `pt`, `abseta`) for a cut on reconstructed V (normally `pt`)
  * `-P rate -v lepN_V` can make a rate plot for a trigger requiring at least `N` leptons with a cut on V (normally `pt`)
  * `-P isorate -v lepN_V --xvar lep_X`  can make an isorate plot of efficiency vs X (e.g. `pt`) for a trigger requiring at least `N` leptons with a cut on V (normally `pt`)

See examples in `scripts/valSuite.sh` for `run-leps` and `plots-leps` 
