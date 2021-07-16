---
title: "Electrons and Photons"
teaching: 10
exercises: 0
questions:
- "How are electrons and photons treated in CMS OpenData?"
objectives:
- "Learn electron member functions for common track-based quantities"
- "Bookmark informational web pages for electrons and photons"
- "Learn member functions for identification and isolation of electrons"
- "Learn member functions for electron detector-related quantities"
keypoints:
- "Quantities such as impact parameters and charge have common member functions."
- "Physics objects in CMS are reconstructed from detector signals and are never 100% certain!"
- "Identification and isolation algorithms are important for reducing fake objects."
- "Member functions for these algorithms are documented on public TWiki pages."
---

Electrons and photons are both reconstructed in the electromagnetic calorimeter in CMS, so they share many common properties and functions.
In POET we will study the `ElecronAnalyzer.cc` and `PhotonAnalyzer.cc`.

## Electron 4-vector and track information

In the loop over the electron collection in `ElectronAnalyzer.cc`, we access elements of the four-vector as shown in the last episode: 
~~~
for (reco::GsfElectronCollection::const_iterator itElec=myelectrons->begin(); itElec!=myelectrons->end(); ++itElec){
    ...
    electron_e.push_back(itElec->energy());
    electron_pt.push_back(itElec->pt());
    ...
}
~~~
{: .language-cpp}

Most charged physics objects are also connected to tracks from the CMS tracking detectors. The charge of the object can be queried directly:
~~~
electron_ch.push_back(itElec->charge());
~~~
{: .language-cpp}

Information from tracks provides other kinematic quantities that are common to multiple types of objects.
Often, the most pertinent information about an object to access from its
associated track is its **impact parameter** with respect to the primary interaction vertex.
Since muons can also be tracked through the muon detectors, we first check if the track is
well-defined, and then access impact parameters in the xy-plane (`dxy` or `d0`) and along
the beam axis (`dz`), as well as their respective uncertainties. 

~~~
math::XYZPoint pv(vertices->begin()->position());  // line 148
...

auto trk = itElec->gsfTrack();
...
electron_dxy.push_back(trk->dxy(pv));
electron_dz.push_back(trk->dz(pv));
electron_dxyError.push_back(trk->d0Error());
electron_dzError.push_back(trk->dzError());
~~~
{: .language-cpp}

Photons, as neutral objects, do not have a direct track link (though displaced track segments may appear from electrons or positrons produced by the photon as it transits the detector material). While the `charge()` method exists for all objects, it is not used in photon analyses. 

## Detector information for identification

The most signicant difference between a list of certain particles from a Monte Carlo generator and a list
of the corresponding physics objects from CMS is likely the inherent uncertainty in the reconstruction.
Selection of "a muon" or "an electron" for analysis requires algorithms designed to separate "real"
objects from "fakes". These are called **identification** algorithms.

Other algorithms are designed to measure the amount of energy deposited near the object, to determine
if it was likely produced near the primary interaction (typically little nearby energy), or from the
decay of a longer-lived particle (typically a lot of nearby energy). These are called **isolation**
algorithms. Many types of isolation algorithms exist to deal with unique physics cases!

Both types of algorithms function using **working points** that are described on a spectrum from
"loose" to "tight". Working points that are "looser" tend to have a high efficiency for accepting
real objects, but perhaps a poor rejection rate for "fake" objects. Working points that are
"tighter" tend to have lower efficiencies for accepting real objects, but much better rejection
rates for "fake" objects. The choice of working point is highly analysis dependent! Some analyses
value efficiency over background rejection, and some analyses are the opposite.

The "standard" identification and isolation algorithm results can be accessed from the physics
object classes.

 * Electrons: [EGamma Public Data (2011 and 2012)](https://twiki.cern.ch/twiki/bin/view/CMSPublic/EgammaPublicData)
 * Photons: [7 TeV/2011 Photon identification](https://cms-physics.web.cern.ch/cms-physics/public/EGM-10-006-pas.pdf), [8 TeV/2012 Photon identification](https://arxiv.org/pdf/1502.02702.pdf)

>Note: current POET implementations of identification working points are appropriate for 2012 data analysis.
{: .testimonial}


Most `reco::<object>` classes contain member functions that return detector-related information. In the
case of electrons, we see this information used as identification criteria:

~~~
bool isLoose = false, isMedium = false, isTight = false;
if ( abs(itElec->eta()) <= 1.479 ) {   
  if ( abs(itElec->deltaEtaSuperClusterTrackAtVtx()) < .007 && abs(itElec->deltaPhiSuperClusterTrackAtVtx()) < .15 && 
       itElec->sigmaIetaIeta() < .01 && itElec->hadronicOverEm() < .12 && 
       abs(trk->dxy(pv)) < .02 && abs(trk->dz(pv)) < .2 && 
       missing_hits <= 1 && passelectronveto==true &&
       abs(1/itElec->ecalEnergy()-1/(itElec->ecalEnergy()/itElec->eSuperClusterOverP()))<.05 &&
       el_pfIso < .15 ){
        
    isLoose = true;
        
    if ( abs(itElec->deltaEtaSuperClusterTrackAtVtx())<.004 && abs(itElec->deltaPhiSuperClusterTrackAtVtx())<.06 && abs(trk->dz(pv))<.1 ){
      isMedium = true;
              
      if (abs(itElec->deltaPhiSuperClusterTrackAtVtx())<.03 && missing_hits<=0 && el_pfIso<.10 ){
        isTight = true;
      }
    }
  }
}
~~~
{: .language-cpp}

Let's break down these criteria:
 * `deltaEta...` and `deltaPhi...` indicate how the electron's trajectory varies between the track and the ECAL cluster,
with smaller variations preferred for the "tightest" quality levels.
 * `sigmaIetaIeta` describes the variance of the ECAL cluster in psuedorapidity ("ieta" is an integer index for this angle).
 * `hadronicOverEm` describes the ratio of HCAL to ECAL energy depositrs, which should be small for good quality electrons.
 * The impact parameters `dxy` and `dz` should also be small for good quality electrons produced in the initial collision.
 * Missing hits are gaps in the trajectory through the inner tracker (shouldn't be any!)
 * The conversion veto is an algorithm that rejects electrons coming from photon conversions in the tracker, which should instead be reconstructed as part of the photon.
 * The criterion using `ecalEnergy` and `eSuperClusterOverP` compares the differences between the electron's energy and momentum measurements, which should be very similar to each other for good electrons. 
 * `el_pfIso` represents how much energy, relative to the electron's, within a cone around the electron comes from other particle-flow candidates. If this value is small the electron is likely "isolated" in the local region.

**Isolation** is computed in similar ways for all physics objects: search for particles in a cone around the object of interest and sum up their energies, subtracting off the energy deposited by pileup particles. This sum divided by the object of interest's transverse momentum is called **relative isolation** and is the most common way to determine whether an object was produced "promptly" in or following the proton-proton collision (ex: electrons from a Z boson decay, or photons from a Higgs boson decay). Relative isolation values will tend to be large for particles that emerged from weak decays of hadrons within jets, or other similar "nonprompt" processes. For electrons, isolation is computed as:

~~~
float el_pfiso = 999;
if (itElec->passingPflowPreselection()) {
  double rho = *(rhoHandle.product());
  double Aeff = effectiveArea0p3cone(itElec->eta());
  auto iso03 = itElec->pfIsolationVariables();
  el_pfIso = (iso03.chargedHadronIso + std::max(0,iso03.neutralHadronIso + iso03.photonIso - rho*Aeff)/itElec->pt();
}
~~~
{: .language-cpp}

Photon isolation and identification are very similar to the formulas for electrons, with different specific criteria. 

{% comment %}


## Generated particle matching -- FIXME

Simulated files also contain information about the generator-level particles that
were propagated into the showering and detector simulations. Physics objects can
be matched to these generated particles spatially.


FIXME TO POET

The AOD2NanoAOD tool sets up several utility functions for matching: `findBestMatch`,
`findBestVisibleMatch`, and `subtractInvisible`. The `findBestMatch` function takes
generated particles (with an automated type `T`) and the 4-vector of a physics
object. It uses angular separation to find the closest generated particle to the
reconstructed particle:

~~~
template <typename T>
int findBestMatch(T& gens, reco::Candidate::LorentzVector& p4) {

  # initial definition of "closest" is really bad
  float minDeltaR = 999.0;
  int idx = -1;

  # loop over the generated particles
  for (auto g = gens.begin(); g != gens.end(); g++) {
    const auto tmp = deltaR(g->p4(), p4);

    # if it's closer, overwrite the definition of "closest"
    if (tmp < minDeltaR) {
      minDeltaR = tmp;
      idx = g - gens.begin();
    }
  }
  return idx; # return the index of the match
}
~~~
{: .language-cpp}

The other utility functions are similar, but correct for generated particles that
decay to neutrinos, which would affect the "visible" 4-vector. 

Loop over selected electrons and use the findBestVisibleMatch function to match it to an "interesting" particle and then to a jet.

FIXME TO POET
~~~
// Match electrons with gen particles and jets
for (auto p = selectedElectrons.begin(); p != selectedElectrons.end(); p++) {
  // Gen particle matching
  auto p4 = p->p4();
  auto idx = findBestVisibleMatch(interestingGenParticles, p4);
  if (idx != -1) {
    auto g = interestingGenParticles.begin() + idx;
    value_gen_pt[value_gen_n] = g->pt();
    value_gen_eta[value_gen_n] = g->eta();
    value_gen_phi[value_gen_n] = g->phi();
    value_gen_mass[value_gen_n] = g->mass();
    value_gen_pdgid[value_gen_n] = g->pdgId();
    value_gen_status[value_gen_n] = g->status();
    value_el_genpartidx[p - selectedElectrons.begin()] = value_gen_n;
    value_gen_n++;
  }

  // Jet matching
  value_el_jetidx[p - selectedElectrons.begin()] = findBestMatch(selectedJets, p4);
}
~~~
{: .language-cpp}

FIXME -- NOTE SIMILARITIES/DIFFERENCES FOR PHOTONS
{% endcomment %}


{% include links.md %}

