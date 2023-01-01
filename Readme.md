
# NLP Tips and Tricks

## Talk slides from NormConf

This repo is a WIP!  I'm moving things into it :) It started from a talk I gave at Normconf on NLP tips and tricks, based on some recent client projects (but I mostly used representative other data in the talk, specifically a dataset of prompts for image gen from Hugging Face).

My talk slides are here:
[https://ghostweather.slides.com/lynncherny/nlp-tips-tricks](https://ghostweather.slides.com/lynncherny/nlp-tips-tricks)
The video (shorter than the slides, I didn't have time to cover it all) is [here](https://www.youtube.com/watch?v=Rs92i_xrBLo&t=2s).


## Deduping and Entity Resolution

You did your entity recognition, and now you have a bunch of entities that overlap -- mispellings, alternate name formats, etc.  What do you do?

Check [here](dedupe_link_text.md) for info on deduping and merging and generally "entity resolution."

## Text Data Cleaning Links

General links on text cleaning for NLP operations.

Page started [here](text_cleaning.md).


## SpaCy and Code For It

SpaCy can be tough to get into, because it's big. This is a list of references, but also code samples for things I want to do to pipelines that I learned through a lot of effort.  Included is how to override a text classifier with rules (e.g., a list of names of porn stars that automatically classify 'nsfw' in a certain context), some examples of fixing up entity errors with token code, and using an NER bert model in a spacy pipeline, which is perhaps unorthodox.

See [spacy_samples](spacy_components.md).

 
## UMAP and text embedding vis

Always plot your text, embedded.  Word2Vec or sentence/doc embeddings work.  You can use this for labeling, for cleaning, for just understanding... and for impressing clients/bosses.

Umap interactive with bokeh [gist](https://gist.github.com/arnicas/78fa9b62e40a6762e1ad4246ce4fc53d)

Related libs: [bulk](https://github.com/koaning/bulk) by Vincent Warmerdam, [umap](https://github.com/lmcinnes/umap) from Leland McInnes, [ThisNotThat](https://github.com/TutteInstitute/thisnotthat) by Leland McInnes and Benoit Hamelin (in progress)


## Weak Supervision

These libs I did not cover in the talk, but they are related to the topics of rules and pattern matching in NLP.

[skweak](https://github.com/NorskRegnesentral/skweak) will work with spaCy.
There is even a spaCy project example.

[Argilla](https://github.com/argilla-io/argilla) (formerly Rubrix): I like the elastic-search-based rule creation process, it's fast.

[Snorkel](https://www.snorkel.org/use-cases/01-spam-tutorial) -- but the project has turned into a company and so the docs are a bit all over



