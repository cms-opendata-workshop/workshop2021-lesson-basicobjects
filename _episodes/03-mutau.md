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

FIXME info about electrons and photons

## Electron 4-vector and track information

FIXME TO POET

In the loop over the electron collection, access the elements of the four-vector: 
~~~
  value_mu_pt[value_mu_n] = it->pt();
  value_mu_eta[value_mu_n] = it->eta();
  value_mu_phi[value_mu_n] = it->phi();
  value_mu_mass[value_mu_n] = it->mass();
~~~
{: .language-cpp}

Many objects are also connected to tracks from the CMS tracking detectors. Information from
tracks provides other kinematic quantities that are common to multiple types of objects.
Often, the most pertinent information about an object to access from its
associated track is its **impact parameter** with respect to the primary interaction vertex.
Since muons can also be tracked through the muon detectors, we first check if the track is
well-defined, and then access impact parameters in the xy-plane (`dxy` or `d0`) and along
the beam axis (`dz`), as well as their respective uncertainties. 

FIXME TO POET

~~~
value_mu_charge[value_mu_n] = it->charge();
auto trk = it->globalTrack(); // muon track

if (trk.isNonnull()) {
   value_mu_dxy[value_mu_n] = trk->dxy(pv);
   value_mu_dz[value_mu_n] = trk->dz(pv);
   value_mu_dxyErr[value_mu_n] = trk->d0Error();
   value_mu_dzErr[value_mu_n] = trk->dzError();
}
~~~
{: .language-cpp}


FIXME -- NOTE SIMILARITIES AND DIFFERENCES FOR TAUs

## Detector information for identification

 * Muons: [SWGuide Muon ID](https://twiki.cern.ch/twiki/bin/view/CMSPublic/SWGuideMuonId)
 * Tau leptons: [Legacy Tau ID Run 1](https://twiki.cern.ch/twiki/bin/view/CMSPublic/WorkBookPFTauTagging#Legacy_Tau_ID_Run_I), [Nutshell Recipe](https://twiki.cern.ch/twiki/bin/view/CMSPublic/NutShellRecipeFor5312AndNewer)

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
for (auto it = muons->begin(); it != muons->end(); it++) {

    // If this muon has isolation quantities...
    if (it->isPFMuon() && it->isPFIsolationValid()) {

       // get the isolation info in a certain cone size:
       auto iso04 = it->pfIsolationR04();

       // and calculate the energy relative to the muon's transverse momentum
       value_mu_pfreliso04all[value_mu_n] = (iso04.sumChargedHadronPt + iso04.sumNeutralHadronEt + iso04.sumPhotonEt)/it->pt();
    }

    // Store the pass/fail decisions about Tight ID
    value_mu_tightid[value_mu_n] = muon::isTightMuon(*it, *vertices->begin());
}
~~~
{: .language-cpp}


The CMS Tau object group relies almost entirely on pre-computed algorithms to determine the
quality of the tau reconstruction and the decay type. Since this object is not stable and has
several decay modes, different combinations of identification and isolation algorithms are
used across different analyses. The TWiki page provides a large table of available algorithms.

In contrast to the muon object, tau algorithm results are typically saved in the AOD files
as their own PFTauDisciminator collections, rather than as part of the tau object class.

~~~
// Get the tau collection
Handle<PFTauCollection> taus;
iEvent.getByLabel(InputTag("hpsPFTauProducer"), taus);

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
for (auto it = taus->begin(); it != taus->end(); it++) {

    // store the tau decay mode
    value_tau_decaymode[value_tau_n] = it->decayMode();

    // Discriminators
    const auto idx = it - taus->begin();
    value_tau_iddecaymode[value_tau_n] = tausDecayMode->operator[](idx).second;
    value_tau_idisoraw[value_tau_n] = tausRawIso->operator[](idx).second;
    value_tau_idisovloose[value_tau_n] = tausVLooseIso->operator[](idx).second;
    // ...etc...
}
~~~
{: .language-cpp}


## Generated particle matching


FIXME -- NOTE VISIBLE MATCH FUNCTIONS FOR TAUS?



{% include links.md %}

