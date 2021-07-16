---
title: "Triggers"
teaching: 5
exercises: 0
questions:
- "How are triggers stored using POET?"
objectives:
- "Learn the basics of the POET trigger analyzer"
keypoints:
- "Trigger paths are stored as a map with names paired to prescale values"
- "4-vector information is stored for objects matching a specific, configurable, trigger filter"
---

You learned in the pre-exercises about selecting trigger paths and determining pre-scale values. Trigger information can be stored
in POET using the `TriggerAnalyzer` and `TriggObjectAnalyzer`.

## Trigger Analyzer

The `TriggerAnalyzer` allows you to store the pass/fail results of certain trigger paths, which can be configured using wildcards. The configuration
also needs to know the names of the trigger collections in the ROOT file. 
~~~
process.mytriggers = cms.EDAnalyzer('TriggerAnalyzer',
                              processName = cms.string("HLT"),
                              #---- These are example triggers for 2012
                              #---- Wildcards * and ? are accepted (with usual meanings)
                              #---- If left empty, all triggers will run              
                              triggerPatterns = cms.vstring("HLT_L2DoubleMu23_NoVertex_v*","HLT_Mu12_v*", "HLT_Photon20_CaloIdVL_v*", "HLT_Ele22_CaloIdL_CaloIsoVL_v*", "HLT_Jet370_NoJetID_v*"), 
                              triggerResults = cms.InputTag("TriggerResults","","HLT"),
                              triggerEvent   = cms.InputTag("hltTriggerSummaryAOD","","HLT")                             
                              )
~~~
{: .language-python}

In `TriggerAnalyzer.cc`, all available triggers are tested against these patterns for matches, and the matches are stored as a string+integer pairs using `std::map<std::string, int>`.
For each trigger that matched one of the requested patterns, the name is stored along with a numerical value that gives the "accept bit" (0 or 1) multiplied by the Level-1 and High-Level trigger prescale vales. This is summarized in the tree's branch title:
~~~
  mtree->Branch("triggermap", &trigmap);
  //second  stores the multiplication acceptbit*L1ps*HLTps 
  //so, if negative, it means that L1ps couldn't be found.
  //look below in the code to understand the specifics
  mtree->GetBranch("triggermap")->SetTitle("first:name of trigger, second: acceptbit*L1ps*HLTps");
~~~
{: .language-cpp}

## Trigger Object Analyzer

It is also possible to store information about specific physics objects that passed a specific trigger "filter". Each trigger path will require objects to pass a wide variety of sequential
filters in order to arrive at the final pass/fail decision for the event. In analyses it is often interesting to know which physics object actually satisfied the requirement of the trigger. For example, a dilepton search using an electron+muon trigger may wish to reconstruct decays using the specific electron that and specific muon that passed the trigger criteria.

In `poet_cfg.py` one trigger "filter" can be configured for `TriggObjectAnalyzer`:

~~~
process.mytrigEvent = cms.EDAnalyzer('TriggObjectAnalyzer',
                                     filterName = cms.string("hltL2DoubleMu23NoVertexL2PreFiltered"),
				    )
~~~
{: .language-python}

In `TriggObjectAnalyzer.cc`, this filter name is used to access a specific list of "keys" and an "object collection" from the trigger summary collection in the ROOT file.
~~~
InputTag trigEventTag("hltTriggerSummaryAOD","","HLT"); //make sure have correct process on MC
//data process=HLT, MC depends, Spring11 is REDIGI311X
Handle<trigger::TriggerEvent> mytrigEvent;
iEvent.getByLabel(trigEventTag,mytrigEvent);

numtrigobj = 0;
trigobj_e.clear();
trigobj_pt.clear();
trigobj_px.clear();
trigobj_py.clear();
trigobj_pz.clear();
trigobj_eta.clear();
trigobj_phi.clear();

trigger::size_type filterIndex = mytrigEvent->filterIndex(edm::InputTag(filterName_,"",trigEventTag.process()));
if(filterIndex<mytrigEvent->sizeFilters()){
  const trigger::Keys& trigKeys = mytrigEvent->filterKeys(filterIndex);
  const trigger::TriggerObjectCollection & trigObjColl(mytrigEvent->getObjects());
~~~
{: .language-cpp}

We can then loop through the list of keys and access the object with that key from the object collection. Storing
this object's basic 4-vector properties allows the analyst to find the electron, muon, tau, jet, etc that is closest to
the trigger object in angular separation.
~~~
  //now loop of the trigger objects passing filter
  for(trigger::Keys::const_iterator keyIt=trigKeys.begin();keyIt!=trigKeys.end();++keyIt){
    const trigger::TriggerObject trigobj = trigObjColl[*keyIt];

    //do what you want with the trigger objects, you have
    //eta,phi,pt,mass,p,px,py,pz,et,energy accessors
    trigobj_e.push_back(trigobj.energy());
    trigobj_pt.push_back(trigobj.pt());
    trigobj_px.push_back(trigobj.px());
    trigobj_py.push_back(trigobj.py());
    trigobj_pz.push_back(trigobj.pz());
    trigobj_eta.push_back(trigobj.eta());
    trigobj_phi.push_back(trigobj.phi());

    numtrigobj=numtrigobj+1;
  }
}//end filter size check
~~~
{: .language-cpp}

For jets, a trigger object matched within the size of the jet cone (perhaps dR < 0.5) would constitute a match. For leptons and photons a smaller separation is typically used,
perhaps dR < 0.2. The exact requirements will be analysis specific!


{% include links.md %}

