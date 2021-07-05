---
title: "Electrons and Photons"
teaching: 15
exercises: 0
questions:
- "How are electrons and photons treated in CMS OpenData?"
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
value_el_pt[value_el_n] = it->pt();
value_el_eta[value_el_n] = it->eta();
value_el_phi[value_el_n] = it->phi();
value_el_mass[value_el_n] = it->mass();
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
value_el_charge[value_el_n] = it->charge();

auto trk = it->gsfTrack();
value_el_dxy[value_el_n] = trk->dxy(pv);
value_el_dz[value_el_n] = trk->dz(pv);
value_el_dxyErr[value_el_n] = trk->d0Error();
value_el_dzErr[value_el_n] = trk->dzError();
~~~
{: .language-cpp}


FIXME -- NOTE SIMILARITIES AND DIFFERENCES FOR PHOTONS

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

 * Electrons: [EGamma Public Data](https://twiki.cern.ch/twiki/bin/view/CMSPublic/EgammaPublicData)
 * Photons: [EGamma Public Data](https://twiki.cern.ch/twiki/bin/view/CMSPublic/EgammaPublicData), [Summary document (see Table 1 for photon ID)](https://cms-physics.web.cern.ch/cms-physics/public/EGM-10-006-pas.pdf)

FIXME -- ADAPT THIS BELOW TO BOTH OBJECTS

Most `reco::<object>` classes contain member functions that return detector-related information. In the
case of electrons and photons, we see this information used as identification criteria:

~~~
value_el_isLoose[value_el_n] = false;
value_el_isMedium[value_el_n] = false;
value_el_isTight[value_el_n] = false;
if ( abs(it->eta()) <= 1.479 ) {
  if ( abs(it->deltaEtaSuperClusterTrackAtVtx())<.007 && abs(it->deltaPhiSuperClusterTrackAtVtx())<.15 &&
       it->sigmaIetaIeta()<.01 && it->hadronicOverEm()<.12 &&
       abs(trk->dxy(pv))<.02 && abs(trk->dz(pv))<.2 &&
       missing_hits<=1 && pfIso<.15 && passelectronveto==true &&
       abs(1/it->ecalEnergy()-1/(it->ecalEnergy()/it->eSuperClusterOverP()))<.05 ){

    value_el_isLoose[value_el_n] = true;

    if ( abs(it->deltaEtaSuperClusterTrackAtVtx())<.004 && abs(it->deltaPhiSuperClusterTrackAtVtx())<.06 && abs(trk->dz(pv))<.1 ){
      value_el_isMedium[value_el_n] = true;

      if (abs(it->deltaPhiSuperClusterTrackAtVtx())<.03 && missing_hits<=0 && pfIso<.10 ){
        value_el_isTight[value_el_n] = true;
      }
    }
  }
}
~~~
{: .language-cpp}

The first two criteria (`deltaEta` and `deltaPh`) indicate how the electron's trajectory varies between the track and the ECAL cluster,
with smaller variations preferred for the "tightest" quality levels. The `sigmaIetaIeta` criterion describes the variance of the ECAL
cluster in psuedorapidity (recall the "ieta" labels on the red LEGO bricks!). There are futher criteria for the ratio of hadronic to 
electromagnetic energy deposits, the track impact parameters, and the difference between the ECAL energy and electron's momentum --
all of which are expected to be small for genuine electrons with well-reconstructed tracks. Finally, a good electron should have very 
few "missing hits" (gaps in the trajectory through the inner tracker), be reasonably isolated from other particle-flow candidates in the
nearby spatial region, and should pass an algorithm that rejects electrons from photon conversion in the tracker. Similar information from 
the detector is used to form the identification criteria for all physics objects. 

## Generated particle matching

Simulated files also contain information about the generator-level particles that
were propagated into the showering and detector simulations. Physics objects can
be matched to these generated particles spatially.

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



{% include links.md %}

