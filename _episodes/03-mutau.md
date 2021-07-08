---
title: "Prep: Muons and Taus"
teaching: 10
exercises: 0
questions:
- "How are muons and taus treated in CMS OpenData?"
objectives:
- "Learn member functions for common track-based quantities"
- "Learn how to connect a physics object with a generator-level particle"
- "Bookmark informational web pages for different objects"
- "Learn member functions for identification and isolation of objects"
- "Learn member functions for detector-related quantities"
- "Practice accessing and saving these quantities"
keypoints:
- "Objects are matched to generator-level particles based on spatial relationships."
- "Other quantities such as impact parameters and charge have common member functions."
- "Physics objects in CMS are reconstructed from detector signals and are never 100% certain!"
- "Identification and isolation algorithms are important for reducing fake objects."
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
neutral hadrons, and energy from photons, all in a cone of radius $\Delta R < 0.3$ or 0.4 around
the muon. Many algorithms also feature a "correction factor" that subtracts average energy expected
from pileup contributions to this cone. Decisions are made by comparing this energy sum to the
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
iEvent.getByLabel(tauInput, mytaus);

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


## Generated particle matching


FIXME -- NOTE VISIBLE MATCH FUNCTIONS FOR TAUS?



{% include links.md %}

