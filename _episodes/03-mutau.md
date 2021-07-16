---
title: "Muons and Taus"
teaching: 10
exercises: 0
questions:
- "How are muons and taus treated in CMS OpenData?"
objectives:
- "Learn member functions for muon track-based quantities"
- "Bookmark informational web pages for different objects"
- "Learn member functions for identification and isolation of muons and taus"
keypoints:
- "Track access may differ, but track-related member functions are common across objects."
- "Physics objects in CMS are reconstructed from detector signals and are never 100% certain!"
- "Muons and taus typically use pre-configured identification and isolation variable member functions."
- "Member functions for these algorithms are documented on public TWiki pages."
---

Muons and tau leptons are have many features that are similar to electrons and photons, but their own unique identification algorithms. In this episode we will be studying `MuonAnalyzer.cc` and `TauAnalyzer.cc`. We explored the muon kinematics member functions in Episode 1, which are identical for all objects. 

CMS TWiki references:
 * Muons: [SWGuide Muon ID](https://twiki.cern.ch/twiki/bin/view/CMSPublic/SWGuideMuonId)
 * Tau leptons: [Legacy Tau ID Run 1](https://twiki.cern.ch/twiki/bin/view/CMSPublic/WorkBookPFTauTagging#Legacy_Tau_ID_Run_I), [Nutshell Recipe](https://twiki.cern.ch/twiki/bin/view/CMSPublic/NutShellRecipeFor5312AndNewer)

## Muon identification and isolation

Muons have a different member functions for accessing the associated track compared to electrons:

~~~
auto trk = itmuon->globalTrack();
if (trk.isNonnull()) {
  muon_dxy.push_back(trk->dxy(pv));
  muon_dz.push_back(trk->dz(pv));
  muon_dxyErr.push_back(trk->d0Error());
  muon_dzErr.push_back(trk->dzError());
  }
~~~
{: .language-cpp}

The CMS Muon object group has created member functions for the identification algorithms that simply
storing pass/fail decisions about the quality of each muon. As shown below, the algorithm depends
on which vertex is being considered as the primary interaction vertex!

Hard processes produce large angles between the final state partons. The final object of interest will be separated from 
the other objects in the event or be "isolated". For instance, an isolated muon might be produced in the decay of a W boson.
In contrast, a non-isolated muon can come from a weak decay inside a jet. 

Muon isolation is calculated from a combination of factors: energy from charged hadrons, energy from
neutral hadrons, and energy from photons, all in a cone of radius dR < 0.3 or 0.4 around
the muon. Many algorithms also feature a "correction factor" that subtracts average energy expected
from pileup contributions to this cone -- we'll explore this in the hands-on exercise. Decisions are made by comparing this energy sum to the
transverse momentum of the muon. 

~~~
if (itmuon->isPFMuon() && itmuon->isPFIsolationValid()) {
  auto iso04 = itmuon->pfIsolationR04();
  muon_pfreliso04all.push_back((iso04.sumChargedHadronPt + iso04.sumNeutralHadronEt + iso04.sumPhotonEt)/itmuon->pt());
}

muon_tightid.push_back(muon::isTightMuon(*itmuon, *vertices->begin()));
~~~
{: .language-cpp}

## Tau identification

The CMS Tau object group relies almost entirely on pre-computed algorithms to determine the
quality of the tau reconstruction and the decay type. Since this object is not stable and has
several decay modes, different combinations of identification and isolation algorithms are
used across different analyses. The TWiki page provides a large table of available algorithms.

In contrast to the muon object, tau algorithm results are typically saved in the AOD files
as their own PFTauDisciminator collections, rather than as part of the tau object class.

~~~
// Get the tau collection (the exact name is given in poet_cfg.py
Handle<reco::PFTauCollection> mytaus;
iEvent.getByLabel(tauInput, mytaus); // tauInput opens "hpsPFTauProducer"

// Get various tau discriminator collections
Handle<PFTauDiscriminator> tausLooseIso, tausVLooseIso, tausMediumIso, tausTightIso,
                           tausDecayMode, tausRawIso;

iEvent.getByLabel(InputTag("hpsPFTauDiscriminationByDecayModeFinding"),tausDecayMode);
iEvent.getByLabel(InputTag("hpsPFTauDiscriminationByRawCombinedIsolationDBSumPtCorr"),tausRawIso);
iEvent.getByLabel(InputTag("hpsPFTauDiscriminationByVLooseCombinedIsolationDBSumPtCorr"),tausVLooseIso);
//...etc...
~~~
{: .language-cpp}

The tau discriminator collections act as pairs, containing the index of the tau and the value
of the discriminant for that tau. Note that the arrays are filled by calls to the individual
discriminant objects, but referencing the vector index of the tau in the main tau collection.

~~~
for (reco::PFTauCollection::const_iterator itTau=mytaus->begin(); itTau!=mytaus->end(); ++itTau){

    // store the tau decay mode
    tau_decaymode.push_back(itTau->decayMode());

    // Discriminators
    const auto idx = itTau - mytaus->begin();
    tau_iddecaymode.push_back(tausDecayMode->operator[](idx).second);
    tau_idisoraw.push_back(tausRawIso->operator[](idx).second);
    tau_idisovloose.push_back(tausVLooseIso->operator[](idx).second);    
    // ...etc...
}
~~~
{: .language-cpp}


## Generator-level particles

In simulation, we can access **generated particles** from the event generation process. 
You can configure `poet_cfg.py` to store information about generated particles with any [Particle Data Group ID numbers](https://pdg.lbl.gov/2021/reviews/rpp2020-rev-monte-carlo-numbering.pdf) and [generator status codes](https://twiki.cern.ch/twiki/bin/view/CMSPublic/WorkBookGenParticleCandidate).
By default, we will store information for final state electrons, muons, and photons, and taus with an intermediate status:
~~~
process.mygenparticle= cms.EDAnalyzer('GenParticleAnalyzer',
          #---- Collect particles with specific "pdgid:status"
          #---- Check PDG ID in the PDG.
          #---- if 0:0, collect them all 
          input_particle = cms.vstring("1:11","1:13","1:22","2:15")
          )
~~~
{: .language-python}

The particles' properties are stored in `GenParticleAnalyzer.cc`. The input collection is not configurable because it is constant across all CMS simulation
samples: "genParticles". 
~~~
Handle<reco::GenParticleCollection> gens;
iEvent.getByLabel("genParticles", gens);
~~~
{: .language-cpp}

We then process the configurable `input_particle` string that was provided from the configuration file. The constructor opens this parameter into a vector of strings called `particle`:
~~~
GenParticleAnalyzer::GenParticleAnalyzer(const edm::ParameterSet& iConfig):
particle(iConfig.getParameter<std::vector<std::string> >("input_particle"))
{
  //now do what ever initialization is needed
  ...code...
}
~~~
{: .language-cpp}

In the `analyze` function we can parse the desired particle/status pairs and check each generated particle against these conditions before storing its kinematic properties, status, and PDG ID into tree branches:
~~~
unsigned int i;
string s1,s2;
std::vector<int> status_parsed;
std::vector<int> pdgId_parsed;
std::string delimiter = ":";

for(i=0;i<particle.size();i++)
{
    //get status and pgdId from configuration
    s1=particle[i].substr(0,particle[i].find(delimiter));
    s2=particle[i].substr(particle[i].find(delimiter)+1,particle[i].size());
    //parse string to int
    status_parsed.push_back(stoi(s1));
    pdgId_parsed.push_back(stoi(s2));
}

if(gens.isValid())
{
  numGenPart=gens->size();
  for (reco::GenParticleCollection::const_iterator itGenPart=gens->begin(); itGenPart!=gens->end(); ++itGenPart)
  {
    //loop trough all particles selected in configuration
    for(i=0;i<particle.size();i++)
    {
      if((status_parsed[i]==itGenPart->status() && pdgId_parsed[i]==itGenPart->pdgId())||(status_parsed[i]==0 && pdgId_parsed[i]==0))
      {
        GenPart_pt.push_back(itGenPart->pt());
        GenPart_eta.push_back(itGenPart->eta());
        GenPart_mass.push_back(itGenPart->mass());
        GenPart_pdgId.push_back(itGenPart->pdgId());
        GenPart_phi.push_back(itGenPart->phi());
        GenPart_status.push_back(itGenPart->status());
        GenPart_px.push_back(itGenPart->px());
        GenPart_py.push_back(itGenPart->py());
        GenPart_pz.push_back(itGenPart->pz());
      }
    }               
  }
}
~~~
{: .language-cpp}

Matching between generated and reconstructed particles is typically done based on spatial relationships. For example, the generated muon (ID = 13) "matched" to a certain reconstructed muon would be the generated muon that has the smallest angular separation from the reconstructed muon. Angular separation is defined as: <img src="https://latex.codecogs.com/svg.image?\Delta&space;R&space;=&space;\sqrt{(\Delta&space;\eta)^2&space;&plus;&space;(\Delta\phi)^2}" title="\Delta R = \sqrt{(\Delta \eta)^2 + (\Delta\phi)^2}" />

An example analysis code that opens a POET file and performs generated particle matching for several types of reconstructed objects is called [MatchingAnalysis.cc](https://github.com/apetkovi1/PhysObjectExtractorTool/blob/master/PhysObjectExtractor/test/MatchingAnalysis.cc).

{% include links.md %}

