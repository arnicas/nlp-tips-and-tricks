<!-- vscode-markdown-toc -->
- [SpaCy Tricks and Examples](#spacy-tricks-and-examples)
  - [Useful Pipeline Components Code](#useful-pipeline-components-code)
    - [Acronym merge with Entity](#acronym-merge-with-entity)
    - [Unpossess Entities](#unpossess-entities)
    - [Remove uncapped "The"](#remove-uncapped-the)
  - [Using Transformers Models in Unorthodox Ways](#using-transformers-models-in-unorthodox-ways)
  - [SpaCy TextCat Models](#spacy-textcat-models)
    - [Examples in the Wild](#examples-in-the-wild)
    - [Official spaCy/Explosion content](#official-spacyexplosion-content)
    - [Overriding a Textcat Classification With Entity Rules](#overriding-a-textcat-classification-with-entity-rules)
  - [Entity Rules and Libs](#entity-rules-and-libs)
  - [Coreference Model](#coreference-model)
    - [Merge Coref Ents in Spacy](#merge-coref-ents-in-spacy)
    - [Pipeline function to add corefs to the Doc](#pipeline-function-to-add-corefs-to-the-doc)
  - [Explosion's SpaCy Resource Links](#explosions-spacy-resource-links)
    - [SpaCy Articles of Use](#spacy-articles-of-use)

<!-- vscode-markdown-toc-config
	numbering=false
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->

# SpaCy Tricks and Examples

## <a name='UsefulPipelineComponentsCode'></a>Useful Pipeline Components Code


### <a name='AcronymmergewithEntity'></a>Acronym merge with Entity

Combine acronym in parens with an entity before it for org names - this looks very hacky, I know:

```
@Language.component("combine_acronym_parens")
def combine_acronym_parens(doc):
    new_ents = []
    for ent in doc.ents:
        # Only check for title if it's a person and not the first token
        if (ent.label_ == "ORG") and len(doc) >= ent.end + 3:
            if (doc[ent.end].text == "(") and (doc[ent.end + 2].text == ")"):
                new_ent = Span(doc, ent.start, ent.end+3, label="ORG")
                new_ents.append(new_ent)
            else:
                new_ents.append(ent)
        else:
            new_ents.append(ent)
    doc.ents = filter_spans(new_ents)  # takes largest enclosing span
    return doc
```

### <a name='UnpossessEntities'></a>Unpossess Entities

Unpossess entities -- remove 's and ' at end of entity. Tokenization issue in default trf spacy model.

```
@Language.component("unpossess_entities")
def unpossess_entities(doc):
    new_ents = []
    for ent in doc.ents:
        if doc[ent.end -1].text == "'s" or doc[ent.end -1].text == "'":
            new_ent = Span(doc, ent.start, ent.end-1, label=ent.label)
            new_ents.append(new_ent)
        else:
            new_ents.append(ent)
    doc.ents = new_ents
    return doc
```


### <a name='RemoveuncappedThe'></a>Remove uncapped "The"


Remove uncapped "the":

```
@Language.component("remove_uncap_the")
def remove_uncap_the(doc):
    new_ents = []
    for ent in doc.ents:
        # Only check for title if it's a person and not the first token
        if ent.label_ == "ORG" and doc[ent.start].text == "the":
            new_ent = Span(doc, ent.start + 1, ent.end, label=ent.label)
            new_ents.append(new_ent)
        else:
            new_ents.append(ent)
    doc.ents = new_ents
    return doc
```


## <a name='UsingTransformersModelsinUnorthodoxWays'></a>Using Transformers Models in Unorthodox Ways

An example of using a pre-trained HF transformers component in spacy pipeline, code from MS's [Presidio Project](https://github.com/microsoft/presidio), a project for de-identifying PII.

This is unorthodox because a trained NER trf model doesn't have the correct final layer for working in a spacy pipeline - it has instead the entity labels. Basically you need to use it as a "stub" component that provides only those labels and info but nothing else.

Code via MS Presidio proj:
```
class TransformersComponent:
    """
    Custom component to use in spacy pipeline.
    Using HaggingFace transformers pretrained models for entity recognition.
    :param pretrained_model_name_or_path: HaggingFace pretrained_model_name_or_path
    """

    def __init__(self, pretrained_model_name_or_path: str) -> None:
        Span.set_extension("confidence_score", default=1.0, force=True)
        tokenizer = AutoTokenizer.from_pretrained(pretrained_model_name_or_path)
        model = AutoModelForTokenClassification.from_pretrained(
            pretrained_model_name_or_path
        )
        self.nlp = pipeline(
            "ner", model=model, tokenizer=tokenizer, aggregation_strategy="simple"
        )

    def __call__(self, doc: Doc) -> Doc:
        """Write transformers results to doc entities."""

        res = self.nlp(doc.text)
        newres = merge_entities(res, doc.text)  # fixes bad merges, at character level
        ents = []
        for d in newres:
            span = doc.char_span(d["start"], d["end"], label=d["entity_group"], alignment_mode='expand')
            if span:
                span._.confidence_score = d["score"]
                ents.append(span)
            else:
                print("no span", d)
        doc.ents = filter_spans(ents)
        return doc

```

How to use it-- this is a complex example that does work, using both a coreference model and then a trained NER model in the same pipeline. The 'combine corefs' component puts a bunch of info on the doc object, and then you use the ner model, and can potentially add a component afterwords that uses both together, e.g., for name resolution.  I haven't gotten that to work well and I'm using crappy heuristics right now for names for one client.

```

@Language.factory(
    "tranformers-ner",
    default-config={"pretrained-model-or-path": "dslim/bert-base-NER"},
)
def create_transformer_model(nlp, name, pretrained_model_name_or_path: str):
    return TransformersComponent(pretrained_model_or_path=pretrained_model_or_path)

nlp = spacy.load("en_coreference_web_trf")
nlp.add_pipe("combine_corefs")
nlp.add_pipe("transformers-ner")
nlp.add_pipe("combine_acronym_parens", after="transformers-ner")
nlp.initialize()
```
Note that after you do the "initialize" you can still add rules.  The rules will get WIPED OUT if you add them before the init, though!  Hard won lessons.


## <a name='SpaCyTextCatModels'></a>SpaCy TextCat Models

### <a name='ExamplesintheWild'></a>Examples in the Wild

* [Example doing it in code loop](https://github.com/jupyter-naas/awesome-notebooks/blob/1a6cc24f0777e5d951ce94aebe0c5dbd470e7880/spaCy/SpaCy_Build_a_sentiment_analysis_model_using_Twitter.ipynb) via juypyter-naas awesome-notebooks
* Tutorial on pipeline method: [Building a Text Classifier with Spacy 3](https://medium.com/analytics-vidhya/building-a-text-classifier-with-spacy-3-0-dd16e9979a) by Phil S.

### <a name='OfficialspaCyExplosioncontent'></a>Official spaCy/Explosion content

You want to make sure you are looking at their [projects](https://github.com/explosion/projects) folder on github.

* spaCy example project in [their demos](https://github.com/explosion/projects/tree/v3/pipelines/textcat_demo) - binary
* [spaCy demo project for multilabel classification](https://github.com/explosion/projects/tree/v3/pipelines/textcat_multilabel_demo)
* Tutorial on [labeling and classifying doc issues](https://github.com/explosion/projects/tree/v3/tutorials/textcat_docs_issues)
* Tutorial on [sentiment classification with textcat](https://github.com/explosion/projects/tree/v3/tutorials/textcat_goemotions)

### <a name='OverridingaTextcatClassificationWithEntityRules'></a>Overriding a Textcat Classification With Entity Rules

Sometimes you want to combine rules with probabilistic models.  Your model might be fragile or incomplete, and you want to be **sure** you catch something with a hard score.
This is how you can set or reset a text classification in a spaCy pipeline. (Note, I wrote this up on [Medium](https://medium.com/@lynn-72328/using-rules-with-a-spacy-textcat-classifier-for-nsfw-content-26ca417611a) too.)

(FYI you might also like Jeremy Jordan's [normconf video](https://www.youtube.com/watch?v=gXe9iXNTuDc) on combining rules and ML models.)

```
from spacy.language import Language

@Language.component("reset_textcat")
def reset_textcat(doc):
    for ent in doc.ents:
        if ent.label_ == "PORN_PERSON":
            doc.cats = {'POSITIVE': 1, 'NEGATIVE': 0}
    return doc

nlp = spacy.load('en_core_web_trf')  # textcat wants transformers pipeline
mytextcat = spacy.load("trained-textcat/model-best")  # load your model
nlp.add_piple("textcat", source=mytextcat)  # trained model here, before entity ruler

# the entity ruler is a set of porn names you've researched - you can load better, but 2 examples:

ruler = nlp.add_pipe("entity_ruler", after="ner", config = {"overwrite_ents": True})

patterns = [{"label": "PORN_PERSON", "pattern": [{"LOWER": "lanna"}, {"LOWER": "rhoades"}]},
            {"label": "PORN_PERSON", "pattern": [{"LOWER": "mia"}, {"LOWER": "khalifa"}]}]
ruler.add_patterns(patterns)  # don't forget to add the rules! 

nlp.add_pipe("reset_textcat", last=True)  # add your override componenent!
```
Now you can test it:

```
doc = nlp("Having coffee with Mia Khalifa in the coffe shop in London, style of Alphonse Mucha")
print("nsfw classification:", doc.cats)
for ent in doc.ents:
    print(ent, ent.label_)
porn classification: {'POSITIVE': 1, 'NEGATIVE': 0}
Mia Khalifa PORN_PERSON
London GPE
Alphonse Mucha PERSON

doc = nlp("Having coffee with Mia Farrow in a coffe shop in London, style of Artgerm")

{'POSITIVE': 0.33414962887763977, 'NEGATIVE':0.6658503413200378}
Mia Farrow PERSON
London GPE
Artgerm PERSON
```

## <a name='EntityRulesandLibs'></a>Entity Rules and Libs

* [Video overview of Entity Linking in SpaCy by Sophie Van Landeghem](Sofie Van Landeghem: Entity linking functionality in spaCy (spaCy IRL 2019).
* [SpaCy docs on the Entity Ruler](https://spacy.io/usage/rule-based-matching#entityruler) -- which is how I overrode names of porn actors in my classifier example.
* [Use NER or a phrase matcher/rule approach](https://support.prodi.gy/t/ner-or-phrasematcher/686)? An FAQ.

Third Party:

* [extractacy](https://github.com/jenojp/extractacy) - a pattern based entity extractor lib for spaCy
* A list of [country patterns for spacy](https://github.com/explosion/prodigy-recipes/blob/master/example-patterns/patterns_countries-GPE.jsonl)
* A [good article](https://towardsdatascience.com/clinical-named-entity-recognition-using-spacy-5ae9c002e86f) by Yu Huang on using rules for medical clinical entity content to update an NER model. "In this method, first a set of medical entities and types was identified, then a spaCy entity ruler model was created and used to automatically generating annotated text dataset for model training and testing, after that a spaCy NER model was created and trained, and finally the same entity ruler model was used to extend the capability of the trained NER model."
* [Implementing Hearst Patterns with SpaCy](https://towardsdatascience.com/implementing-hearst-patterns-with-spacy-216e585f61f8).

## <a name='CoreferenceModel'></a>Coreference Model

Still in experimental, it seems to me pretty good. Stay tuned for more on using this. Blog post about it [here](https://explosion.ai/blog/coref).

FYI, you need to look for it in spacy-experimental, under ["releases."](https://github.com/explosion/spacy-experimental/releases)

### <a name='MergeCorefEntsinSpacy'></a>Merge Coref Ents in Spacy

Merging coreference entities in spacy (a fast hack but useful: "This code snippet shows how to use spaCy's coreference resolution component to resolve references in text. The function shown below is a simple approach and does not support cases like overlapping spans."): [link](https://gist.github.com/thomashacker/b5dd6042c092e0a22c2b9243a64a2466)

### <a name='PipelinefunctiontoaddcorefstotheDoc'></a>Pipeline function to add corefs to the Doc

A function to add all the corefs from the experimental coref model to the doc in a pipeline

```
@Language.component("combine_corefs")
def get_corefs(doc):
    if not Doc.has_extension("corefs"):
        Doc.set_extension("corefs", default=[], force=True)
    clusters = []
    for key, vals in doc.spans.items():
        cluster = []
        for val in vals:
            cluster.append([val.text, val.start, val.end])
        clusters.append(cluster)
    doc._.corefs = clusters
    return doc
```

## <a name='SpaCyResourceLinks'></a>Explosion's SpaCy Resource Links


* [Website](https://spacy.io/usage)
* [Github Discussions](https://github.com/explosion/spaCy/discussions) (this is key! with tags)
* [Videos with lots of tutorial material!](https://www.youtube.com/@ExplosionAI/videos) - both mainstream spaCy and prodigy
* The gigantic scary [flowchart of NER workflow decision points](https://github.com/explosion/assets/blob/main/Prodigy/Prodigy_NER_flowchart_v2_0_0_light.pdf) which is very useful
* [Prodigy](https://prodi.gy/), their customizable labeling tool with an active Support forum and integration with spaCy
* The [Advanced SpaCy online web course](https://course.spacy.io/en/) (which I should redo)
* Tailored analysis/pipelines and [consulting help](https://explosion.ai/custom-solutions) -- this is a thing I would use myself, they are nice (link)


### <a name='SpaCyarticlesofusetome'></a>SpaCy Articles of Use

* [Making the Most of Spacy's Rule-Based Matcher](https://pmbaumgartner.github.io/blog/spacy-rule-based-matcher-workflow/), by Peter Baumgartner