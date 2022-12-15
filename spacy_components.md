 

# SpaCy Tricks and Examples


Merging coreference entities in spacy (a fast hack but useful: "This code snippet shows how to use spaCy's coreference resolution component to resolve references in text. The function shown below is a simple approach and does not support cases like overlapping spans."): [link](https://gist.github.com/thomashacker/b5dd6042c092e0a22c2b9243a64a2466)


Way more coming..!

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





Unpossess entities -- remove 's and ' at end of entity:

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

Transformers Component in spacy pipeline, code from MS's [Presidio Project](https://github.com/microsoft/presidio), for de-identifying PII:

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

How to use it:




### SpaCy Resource Links


* [Website](https://spacy.io/usage)
* [Github Discussions](https://github.com/explosion/spaCy/discussions) (this is key! with tags)
* [Videos with lots of tutorial material](https://www.youtube.com/@ExplosionAI/videos)
* The gigantic scary [flowchart of NER workflow decision points](https://github.com/explosion/assets/blob/main/Prodigy/Prodigy_NER_flowchart_v2_0_0_light.pdf) which is very useful
* [Prodigy](https://prodi.gy/), their customizable labeling tool with an active Support forum and integration with spaCy
* The [Advanced SpaCy online web course](https://course.spacy.io/en/) (which I should redo)
* Tailored analysis/pipelines and [consulting help](https://explosion.ai/custom-solutions) -- this is a thing I would use myself, they are nice (link)