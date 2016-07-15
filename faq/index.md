---
layout: page
title: faq
permalink: /faq/
---


**Q.** Why does modifying the same notebook in nteract create an ipynb file that
   differs from the one created by the Jupyter Notebook?

**A.** The ipynb file format is a JSON based file format.  JSON does not guarantee
   the order of keys.  The order of keys is dependent on the implementation of
   the JSON parser being used.  Jupyter notebook uses Python's JSON parser
   while nteract uses V8's Javascript JSON parser.  If you want to diff two
   notebook files properly, you'll need to use a JSON diffing tool.
