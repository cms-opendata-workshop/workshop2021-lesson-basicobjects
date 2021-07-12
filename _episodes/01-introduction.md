---
title: "Prep: CMS Physics Objects"
teaching: 10
questions:
- "What are the main CMS Physics Objects and how do I access them?"
objectives:
- "Identify CMS physics objects"
- "Identify object code collections in AOD files"
- "Access an object code collection"
- "Learn member functions for standard momentum-energy vectors"
- "Learn member functions for common track-based quantities"
keypoints:
- "CMS physics objects include: muons, electrons, taus, photons, and jets." 
- "Missing transverse momentum is derived from physics objects (negative vector sum)."
- "Objects are stored in separate collections in the AOD files"
- "Objects can be accessed one-by-one via a for loop"
- "Physics objects in CMS inherit common member functions for the 4-vector quantities
   of transverse momentum, polar/azimuthal angles, and mass/energy."
---


CMS uses the phrase "physics objects" to speak broadly about particles that can be identified via 
signals from the CMS detector. A physics object typically began as a single particle, but may have 
showered into many particles as it traversed the detector. The principle physics objects are:

*   muons
*   electrons
*   taus
*   photons
*   jets
*   missing transverse momentum

## Viewing modules in a data file

CMS AOD files store all physics objects of the same type in "modules". These modules are 
data structures that act as containers for multiple instances of the same C++ class. The `edmDumpEventContent` command
show below lists all the modules in a file. As an example,
the "muons" module in AOD contains many `reco::Muon` objects (one per muon in the event). 

~~~
$ edmDumpEventContent root://eospublic.cern.ch//eos/opendata/cms/MonteCarlo2012/Summer12_DR53X/TTbar_8TeV-Madspin_aMCatNLO-herwig/AODSIM/PU_S10_START53_V19-v2/00000/000A9D3F-CE4C-E311-84F8-001E673969D2.root
~~~
{: .language-bash}

~~~
Type                                  Module                      Label             Process
----------------------------------------------------------------------------------------------
...
vector<reco::Muon>                    "muons"                     ""                "RECO"
vector<reco::Muon>                    "muonsFromCosmics"          ""                "RECO"
vector<reco::Muon>                    "muonsFromCosmics1Leg"      ""                "RECO"
...
vector<reco::Track>                   "cosmicMuons"               ""                "RECO"
vector<reco::Track>                   "cosmicMuons1Leg"           ""                "RECO"
vector<reco::Track>                   "globalCosmicMuons"         ""                "RECO"
vector<reco::Track>                   "globalCosmicMuons1Leg"     ""                "RECO"
vector<reco::Track>                   "globalMuons"               ""                "RECO"
vector<reco::Track>                   "refittedStandAloneMuons"   ""                "RECO"
vector<reco::Track>                   "standAloneMuons"           ""                "RECO"
vector<reco::Track>                   "refittedStandAloneMuons"   "UpdatedAtVtx"    "RECO"
vector<reco::Track>                   "standAloneMuons"           "UpdatedAtVtx"    "RECO"
vector<reco::Track>                   "tevMuons"                  "default"         "RECO"
vector<reco::Track>                   "tevMuons"                  "dyt"             "RECO"
vector<reco::Track>                   "tevMuons"                  "firstHit"        "RECO"
vector<reco::Track>                   "tevMuons"                  "picky"           "RECO"

~~~
{: .output}

Note that this file also contains many other muon-related modules: two modules of `reco::Muon` muons 
from cosmic-ray events, and many modules of `reco::Track` objects that give lower-level tracking
information for muons. As you can see, the AOD file contains MANY modules, and not all of them are related directly to 
physics objects. Other important modules might include:

*   Tracks and vertices
*   Calorimeter clusters and other detector information
*   Particle Flow candidates
*   Triggers
*   Pileup (info about multiple collisions from the same pp crossing)
*   Generator-level information (in simulation) 

## Opening a module

>## Setup
>The [PhysObjectExtractorTool](https://github.com/cms-legacydata-analyses/PhysObjectExtractorTool) (POET) 
>repository is the example we will use for accessing information from AOD files. If you have not already done so, 
>please check out the "dummyworkshop" branch of this repository:
>
>~~~
>$ cd ~/CMSSW_5_3_32/src/
>$ cmsenv
>$ git clone -b dummyworkshop git://github.com/cms-legacydata-analyses/PhysObjectExtractorTool.git 
>$ cd PhysObjectExtractorTool
>$ scram b
>~~~
>{: .language-bash}
{: .prereq}

In the various source code files for this tool, found in `PhysObjectExtractor/src/`, the definitions of different classes are included. Continuing with muons as the example, we include the following in `PhysObjectExtractor/src/MuonAnalyzer.cc`:
~~~
#include "DataFormats/MuonReco/interface/Muon.h"
#include "DataFormats/MuonReco/interface/MuonFwd.h"
#include "DataFormats/MuonReco/interface/MuonSelectors.h"
~~~
{: .language-cpp}

You learned about the EDAnalyzer class in the pre-exercises. The POET is an EDAnalyzer. 
The "analyze" function of an EDAnalyzer is performed once per event. Muons can be accessed like this:

~~~
void
MuonAnalyzer::analyze(const edm::Event &iEvent, const edm::EventSetup &iSetup)
{

  using namespace edm;
  using namespace std;

  Handle<reco::MuonCollection> mymuons;
  iEvent.getByLabel(muonInput, mymuons);
~~~ 
{: .language-cpp}

The `edm::InputTag` object `muonInput` is defined in `python/poet_cfg.py` to be "muons", which will point to one of the muon collections we saw in the event content above. The result of the `getByLabel` command is a variable called "mymuons" which is a collection of all the muon objects. 
Collection classes are generally constructed as std::vectors. We can 
quickly access create a loop to access 
individual muons:

~~~
for (reco::MuonCollection::const_iterator itmuon=mymuons->begin(); itmuon!=mymuons->end(); ++itmuon){
    if (itmuon->pt() > mu_min_pt) {
        // do things here, see below!
    }
}
~~~
{: .language-cpp}

## Accessing basic kinematic quantities

Many of the most important kinematic quantities defining a physics object are accessed in a
common way across all the objects. All objects have associated energy-momentum vectors, typically
constructed using **transverse momentum, pseudorapdity, azimuthal angle, and
mass or energy**.

In `MuonAnalyzer.cc` the muon four-vector elements are accessed as shown below. The
values for each muon are stored into an array, which will become a branch in a ROOT TTree. 

~~~
for (reco::MuonCollection::const_iterator itmuon=mymuons->begin(); itmuon!=mymuons->end(); ++itmuon){
  if (itmuon->pt() > mu_min_pt) {

    muon_e.push_back(itmuon->energy());
    muon_pt.push_back(itmuon->pt());
    muon_eta.push_back(itmuon->eta());
    muon_phi.push_back(itmuon->phi());

    muon_px.push_back(itmuon->px());
    muon_py.push_back(itmuon->py());
    muon_pz.push_back(itmuon->pz());

    muon_mass.push_back(itmuon->mass());

}
~~~
{: .language-cpp}

You will see the same type of kinetmatic member functions in all the different analyzers in the `src/` folder!

{% include links.md %}
