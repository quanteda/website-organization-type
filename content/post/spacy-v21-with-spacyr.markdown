---
title: "Using spaCy v2.1 with spacyr"
author: "Akitaka Matsuo"
date: '2019-03-28'
tags: ["blog", "spacyr"]
categories: ["R"]
---



A major update of [**spaCy**](https://spacy.io) (v2.1) was [released](https://github.com/explosion/spaCy/releases/tag/v2.1.0) recently. **spaCy** is one of the best and fastest tools for tokenization, part-of-speech tagging, dependency parsing, and entity recognition. In this post, I will discuss how it works with our [**spacyr**](https://spacyr.quanteda.io) package along with some tips on having multiple versions of **spaCy** using conda environments.

## Good news: It works

Our package **spacyr** is an R wrapper to the **spaCy** Python library. To work with the **spacyr** package, users have to prepare a Python environment with  **spaCy** installed. This may not be difficult if you are familiar with both R and Python, but that may not be necessarily the case for R-only users. To help such users, we implemented a function called `spacy_install()` which provides a one-step solution.

The gist of what the function does is:

1. Create a new conda environment (with new Python executable)
2. pip-install spacy in the new environment 

The implementation of the function owes a lot to a similar function in the [**tensorflow**](https://www.tensorflow.org) package. If everything works as intended, the installation process is truly one-step. And for Mac and Linux users, `spacy_install()` even downloads [Miniconda](https://docs.conda.io/en/latest/miniconda.html) when it does not exist in the system. This seems to be working for most cases. However, as you can see from the [issues](https://github.com/quanteda/spacyr/issues?utf8=%E2%9C%93&q=is%3Aissue) in our GitHub repository, some users have troubles in installing it (see for example [this](https://github.com/quanteda/spacyr/issues/156) issue). Typically, the problem is caused by not having installed a C++ compiler, as some operating systems (pretty much all except Linux) do not include one by default. 

Assuming that you have an old **spaCy** installed through **spacyr**, you can try updating to the new **spaCy** by our other function:


```r
spacy_upgrade()
```

In my environment (Mac OS Mojave), it did not work and the installation stopped in the middle. The error was fixable^[The error message I got was: *ImportError: Something is wrong with the numpy installation. While importing we detected an older version of numpy in ['.../anaconda/envs/spacy_condaenv/lib/python3.6/site-packages/numpy']. One method of fixing this is to repeatedly uninstall numpy until none is found, then reinstall this version.*], but it requires some work in console and does not fit to the purpose of our one-step solution through `spacy_install()`.

So I decided to use an easy solution (which I recoomend if you experience the same issue): Delete the current conda environment and re-install


```r
spacy_uninstall() 
spacy_install()
```

Once we did this on macOS, everything worked fine:


```r
library(spacyr)
spacy_initialize(model = "en_core_web_sm")
## Finding a python executable with spaCy installed...
## spaCy (language model: en_core_web_sm) is installed in C:\Users\Mayazure\AppData\Local\Programs\Python\Python37\python.exe
## successfully initialized (spaCy Version: 2.3.2, language model: en_core_web_sm)
## (python options: type = "python_executable", value = "C:\Users\Mayazure\AppData\Local\Programs\Python\Python37\python.exe")
spacy_parse("hello world")
##   doc_id sentence_id token_id token lemma   pos entity
## 1  text1           1        1 hello hello  INTJ       
## 2  text1           1        2 world world PROPN
```

If you have your original environment (e.g. a custom language model you trained) and do not want to mess up the setup, you can test a new version by creating another environment as described below.


## Another good news: Tokenization is faster and you can feel it in spacyr

The tokenization is really fast in **spaCy** v2.1. That can be seen from `spacy_tokenize()`. The benchmark comparison looks like this:


```
## Unit: milliseconds
##     expr       min        lq      mean    median        uq       max neval
##   v2.1.3  134.6249  143.3171  149.7733  151.7219  155.2604  163.8899    10
##  v2.0.18 1472.2077 1487.7420 1536.3897 1496.1156 1523.6667 1869.4669    10
```

This is based on tokenizing a fairly small corpus of 14 documents containing a total of about 54,000 words (the `quanteda::data_corpus_irishbudget2010` corpus).  Due to the impossiblity of unloading Python (see the last section of this post), each benchmark had to be run in a separate R session and combined afterwards. (See this [gist](https://gist.github.com/amatsuo/7f64299310a110bd8158e3c2b262ff0b) for details.)

The difference is massive: **spaCy** v2.1 is about **10 (!) times faster** in tokenization called from **spacyr**.

So this is great.  However, there is a caveat in this performance gain. **`spacy_tokenize()` is fast under limited conditions.** By default, **spaCy** has [four pipeline components](https://spacy.io/usage/processing-pipelines): `tokenizer`, `tagger`, `parser` and `ner`. In version v2.1, the tokenizer became really, really fast. So, if you run only `tokenizer`, which is the first component of the pipeline (i.e. run `spacy_tokenize()` with all default options), it is very fast. However, if you need to conduct more feature rich tokenization (e.g. `spacy_tokenize(remove_numbers = TRUE)`), the later components of the pipeline have to be run and it will take longer to finish.


```r
data(data_corpus_irishbudget2010, package = "quanteda.textmodels")
microbenchmark::microbenchmark(
    "remove_numbers = TRUE" = spacy_tokenize(data_corpus_irishbudget2010, remove_numbers = TRUE),
    "remove_numbers = FALSE" = spacy_tokenize(data_corpus_irishbudget2010),
    times = 1
)
## Unit: milliseconds
##                    expr       min        lq      mean    median        uq
##   remove_numbers = TRUE 3873.1447 3873.1447 3873.1447 3873.1447 3873.1447
##  remove_numbers = FALSE  106.2504  106.2504  106.2504  106.2504  106.2504
##        max neval
##  3873.1447     1
##   106.2504     1
```

(I didn't check whether or not this slowdown is caused by our code in either R or Python.  We will test this more thoroughly in the future.)


## A note on having multiple versions of spaCy in your spacyr installation

The current setup of `spacy_install()` creates a new conda environment isolated from other environments, and thus, if you want to have multiple versions of **spaCy**, this is easily possible.

One thing to remember for doing that is **you need to restart R when switching from one spaCy to another**. That is because of the difficulty of unloading the Python environment.  In the backend, we use the wonderful [**reticulate**](https://github.com/rstudio/reticulate/) package by RStudio for seamless integration of R with Python. The developer of the package [has made it clear](https://github.com/rstudio/reticulate/issues/27#issuecomment-288728371) that unloading Python is technically difficult and reticulate does not support that.^[spacyr has a function called `spacy_finalize()`. This function deletes all objects created by **spaCy** in the Python space but does not delete the Python space itself.]

Having said that, here is the way to install two versions of spaCy:


```r
library(spacyr)

## install the latest version of spaCy (will be installed in "spacy_condaenv")
spacy_install()

## install an older version of spaCy (in an enviroment "spacy_old")
spacy_install(version = "2.0.18", envname = "spacy_old")
```

To use these environments, you can specify the version when you call `spacy_initialize()`.


```r
## to use latest version
spacy_initialize(refresh_settings = TRUE)

## to use the older version
spacy_initialize(condaenv = "spacy_old", refresh_settings = TRUE)
```

The first line `spacy_initialize(refresh_settings = TRUE)` will use `spacy_condaenv` as that's the first thing **spacyr** will check when initializing **spaCy**. **spacyr** searches possible locations of **spaCy** installation when `spacy_initialize()` is called for the first time (not in this session, but in the entire history). After successfully initializing **spaCy**, **spacyr** will remember the location and use the same setting thereafter. Now that you are switching between two environments, you need to have **spacyr** ignoring the saved setting. `refresh_settings = TRUE` in `spacy_initialize()` will do the job.

## Summary

In this post, I have discussed a few features of **spacyr** related to the new release of **spaCy**. In summary:

1. **spaCy** v2.1 works with **spacyr**;
2. **spaCy** tokenization is much faster in v2.1 than the tokenization in previous versions; and
3. **spacyr** can help you to set up multiple Python environments.


