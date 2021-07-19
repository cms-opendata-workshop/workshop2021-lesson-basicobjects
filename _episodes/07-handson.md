---
title: "Basic objects hands-on"
teaching: 0
exercises: 40
questions:
- "How can I navigate the physics object references to compute identification criteria?"
- "How can I separate events with and without invisible particles?"
objectives:
- "Practice expanding identification criteria beyond POET defaults."
- "Practice interacting with ROOT file output from POET."
keypoints:
- "All physics objects have multiple identification and isolation schemes."
- "POET implements the most common identification and isolation criteria used in analyses."
- "MET exists in all events, but significant differences can be seen between samples with and without real MET."
---

Choose your exercise! The first several exercises all relate to manipulating identification criteria for muons, taus, or jets. Please complete one of them.

>## Exercise 1 option A: add alternate muon IDs and isolation corrections
>
>Using the documentation on the TWiki page:
> * adjust the 0.4-cone muon isolation calculation to apply the "DeltaBeta" pileup correction.
> * add the pass/fail information about the Loose identification working point.
> * try to recreate the Tight identification working point from detector information criteria!
>
>> ## Solution:
>> The DeltaBeta correction for pileup involves subtracting off half of the pileup contribution
>> that can be accessed from the "iso04" object already being used:
>>~~~
>>if (itmuon->isPFMuon() && itmuon->isPFIsolationValid()) {
>>  auto iso04 = itmuon->pfIsolationR04();
>>  muon_pfreliso04all.push_back((iso04.sumChargedHadronPt + iso04.sumNeutralHadronEt + iso04.sumPhotonEt - 0.5*iso04.sumPUPt)/itmuon->pt());
>>~~~
>>{: .language-cpp}
>>
>> To add new variables we need to check four code locations: declarations, branches, vector clearing, and vector filling. 
>> You might add Loose ID beneath the existing Tight and Soft IDs in each section:
>>~~~
>>std::vector<float> muon_softid;
>>std::vector<float> muon_looseid;
>>~~~
>>{: .language-cpp}
>>~~~
>>mtree->Branch("muon_softid",&muon_softid);
>>mtree->GetBranch("muon_softid")->SetTitle("soft cut-based ID");
>>mtree->Branch("muon_looseid",&muon_looseid);
>>mtree->GetBranch("muon_looseid")->SetTitle("loose cut-based ID");
>>~~~
>>{: .language-cpp}
>>~~~
>>muon_softid.clear();
>>muon_looseid.clear();
>>~~~
>>{: .language-cpp}
>>~~~
>>muon_softid.push_back(muon::isSoftMuon(*itmuon, *vertices->begin()));
>>muon_looseid.push_back(muon::isLooseMuon(*itmuon));
>>~~~
>>{: .language-cpp}
>>
>> The TWiki also gives the member functions needed to reconstruct the muon ID. We can see from the built-in tightID method that a 
>> vertex is needed for some of the criteria: `muon::isTightMuon(*it, *vertices->begin())`. To learn more about the
>> vertex collection you can refer to `VertexAnalyzer.cc`.
>> ~~~
>> std::vector<bool> muon_isTightByHand;
>>
>> if( it->isGlobalMuon() && it->isPFMuon() && 
>>     it->globalTrack()->normalizedChi2() < 10. && it->globalTrack()->hitPattern().numberOfValidMuonHits() > 0 &&
>>     it->numberOfMatchedStations() > 1 && 
>>     fabs(it->muonBestTrack()->dxy(vertices->begin()->position())) < 0.2 && fabs(it->muonBestTrack()->dz(vertex->position())) < 0.5 &&
>>     it->innerTrack()->hitPattern().numberOfValidPixelHits() > 0 && it->innerTrack()->hitPattern().trackerLayersWithMeasurement() > 5)
>>    {
>>      muon_isTightByHand.push_back(true);
>>    }
>> ~~~
>>{: .language-cpp}
>{: .solution}
{: .challenge}

>## Exercise 1 option B: add alternate tau IDs
>
>Many other tau discriminants exist. Based on information from the TWiki, 
>save the values for some discriminants that are based on multivariate analysis techniques.
>
>> ## Solution:
>> The TWiki describes Loose/Medium/Tight ID levels for a "IsolationMVA" and "IsolationMVA2" algorithms.
>> They can be accessed like the other tau IDs, but you might need to refer to the output of `edmDumpEventContent` 
>> to find the exact form of the InputTag name. 
>>
>> Add declarations:
>>~~~
>>std::vector<bool> tau_idisoMVA2loose;
>>std::vector<bool> tau_idisoMVA2tight;
>>~~~
>>{: .language-cpp}
>> Add branches:
>>~~~
>>mtree->Branch("tau_idisoMVA2loose",&tau_idisoMVA2loose);
>>mtree->GetBranch("tau_idisoMVA2loose")->SetTitle("tau id loose isolation from MVA2");
>> // ...etc for other ID...
>>~~~
>>{: .language-cpp}
>> Create handles and get the information from the input file:
>>~~~
>>Handle<PFTauCollection> taus;
>>iEvent.getByLabel(InputTag("hpsPFTauProducer"), taus);
>>
>>Handle<PFTauDiscriminator> tausLooseIso, tausVLooseIso, tausMediumIso, tausTightIso,
>>                           tausDecayMode, tausRawIso, tausTightEleRej,
>>			     tausTightMuonRej, tausLooseIsoMVA2, tausTightIsoMVA2;
>>
>> // new things only
>>iEvent.getByLabel(InputTag("hpsPFTauDiscriminationByLooseIsolationMVA2"),tausLooseIsoMVA2);
>>iEvent.getByLabel(InputTag("hpsPFTauDiscriminationByTightIsolationMVA2"),tausTightIsoMVA2);
>>~~~
>>{: .language-cpp}
>>Clear the vectors at the beginning of each event:
>>~~~
>>tau_idisoMVA2loose.clear()
>>tau_idisoMVA2tight.clear()
>>~~~
>>{: .language-cpp}
>> And finally, access the discriminator from the second element of the pair:
>>~~~
>>tau_idisoMVA2loose.push_back(tausLooseIsoMVA2->operator[](idx).second);
>>tau_idisoMVA2tight.push_back(tausTightIsoMVA2->operator[](idx).second);
>>~~~
>>{: .language-cpp}
>{: .solution}
{: .challenge}

>## Exercise 1 option C: apply noise jet ID
>
>Use the [cms-sw github repository](https://github.com/cms-sw/cmssw/tree/CMSSW_5_3_X/DataFormats/PatCandidates/) to learn the methods available for pat::Jets 
>(hint: the header file is included from `PatJetAnalyzer.cc`). Implement the jet ID and **reject** jets that do not pass. Rejection means that information
>about these jets will not be stored in any of the tree branches.
>
>>## Solution
>>The header file we need is for particle-flow jets: `interface/Jet.h` from the link given. It shows many functions like this:
>>~~~
>>/// chargedHadronEnergyFraction (relative to uncorrected jet energy)
>>float chargedHadronEnergyFraction() const {return chargedHadronEnergy()/((jecSetsAvailable() ? jecFactor(0) : 1.)*energy());}
>>~~~
>>{: .language-cpp}
>>These functions give the energy from a certain type of particle flow candidate as a fraction of the jet's total energy. We can apply the
>>conditions given to reject jets from noise at the same time we apply a momentum threshold:
>>~~~
>>for (std::vector<pat::Jet>::const_iterator itjet=myjets->begin(); itjet!=myjets->end(); ++itjet){
>>  if (itjet->chargedHadronEnergyFraction() > 0 && itjet->neutralHadronEnergyFraction() < 1.0 &&
>>      itjet->electronEnergyFraction() < 1.0 && itjet->photonEnergyFraction() < 1.0){
>>
>>    // calculate things on jets
>>  }
>>}
>>~~~
>>{: .language-cpp}
>{: .solution}
{: .challenge}

>## Exercise 2: real and fake MET
>
>Compile all your changes to POET so far and run 400 events from two different simulation samples. 
>One test file contains top quark pair events, so some events will have leptonic decays that include neutrinos
>and some events will not. The other test file contains Drell-Yan events without neutrinos.
>Review TTree::Draw from the pre-exercises -- can you draw histograms of MET versus MET significance 
>and infer which events have leptonic decays? 
>
> ~~~
> $ scram b
> $ # edit python/poet_cfg.py to run over 400 events from the ttbar simulation test file.
> $ cmsRun python/poet_cfg.py
> $ # edit python/poet_cfg.py to use this input file: root://eospublic.cern.ch//eos/opendata/cms/MonteCarlo2012/Summer12_DR53X/DYJetsToLL_M-50_TuneZ2Star_8TeV-madgraph-tarball/AODSIM/PU_RD1_START53_V7N-v1/20000/003063B7-4CCF-E211-9FED-003048D46124.root, and to save a file called myoutput_DY.root
> $ cmsRun python/poet_cfg.py
> $ root -l myoutput.root
> [0] TTree *ttbar = (TTree*)_file0->Get("mymets/Events");
> [1] TFile *_file1 = TFile::Open("myoutput_DY.root");
> [2] TTree *dy = (TTree*)_file1->Get("mymets/Events");
> [3] ttbar->Draw("...a branch name...", "...any cuts go here...", "norm")
> [4] dy->Draw("...a branch name...", "...any cuts go here...", "norm pe same")
> ~~~
> {: .language-bash}
>
>>## Solution
>>
>>The difference between the Drell-Yan events with primarily fake MET and the top pair events with primarily genuine MET
>>can be seen by drawing `MET_pt` or by drawing `MET_significance`. In both distributions the Drell-Yan events have 
>>smaller values than the top pair events.
>>
>>![width=0.5](../assets/img/DYvsTT_MET.png) ![](../assets/img/DYvsTT_signif.png)
>{: .solution}
{: .challenge}

{% include links.md %}
