---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "12 symptoms of a Hidden Technical Debt in your ML project"
subtitle: ""
summary: "How to identify a technical debt and how to pay it off early"
authors: ["kirill-vasin"]
tags: []
categories: []
date: 2021-07-10
lastmod: 2021-07-10
featured: true
draft: false
toc: true
aliases:
  - /post/technical_debt_symptoms/


# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---
## Introduction
About a year ago I stumbled upon a paper called [_"Machine Learning: The High-Interest Credit Card of Technical Debt"_](https://research.google/pubs/pub43146/) written by brilliant engineers from Google in 2014 (also, it has an older version with practically identical contents - [_"Hidden Technical Debt in Machine Learning Systems"_](https://papers.nips.cc/paper/5656-hidden-technical-debt-in-machine-learning-systems.pdf)). I found ideas from there very practical. Every time I come back to this paper, it offers me a fresh perspective on my current projects. So, I decided to rethink this paper in a form of a list of symptoms which can be used to assess a current or planned level of a technical debt in ML-project and create an action plan suitable for a corresponding situation.

I added some comments using *italics* and proposed several solutions (💡) which were not considered in the paper.

## What is a technical debt
Technical debt is a metaphor that refers to an amount of work your team would have to do in a future to provide a quick solution right here right now.
{{< figure src="technical_debt.jpg" title="Guys will have a hard time when they decide to hang TV on another wall">}}

**Technical debt is not a curse** and it is absolutely normal to increase it on the early stage of a new project to deliver results faster. Yet, it is very useful to have some guidelines helping to identify where technical debt can emerge in order to make informed decisions.

## Symptoms of ML-Project Technical Debt
* [Your model relies on many data sources](#-your-model-relies-on-many-data-sources)
* [Your model affects it's own input data](#-your-model-affects-its-own-input-data)
* [Your prediction service has undeclared consumers](#-your-prediction-service-has-undeclared-consumers-aka-visibility-debt)
* [Your model relies on unstable data source](#-your-model-relies-on-unstable-data-source-eg-another-model)
* [You are using features with slim to none performance impact](#-you-are-using-features-with-slim-to-none-performance-impact)
* [You have a lot of “glue code” because of a specific ML-package](#-you-have-a-lot-of-glue-code-because-of-a-specific-ml-package)
* [Data preparation stages turned into pipeline jungles](#-data-preparation-stages-turned-into-pipeline-jungles)
* [You are mixing dead experimental code with a working one](#-you-are-mixing-dead-experimental-code-with-a-working-one)
* [Your configuration files are very complex](#-your-configuration-files-are-very-complex)
* [You have chosen a threshold for your model manually](#-you-have-chosen-a-threshold-for-your-model-manually)
* [Your model relies on non-causal correlations](#-your-model-relies-on-non-causal-correlations)
* [Your ML-system monitoring and testing require improvements](#-your-ml-system-monitoring-and-testing-require-improvements)

### :question_mark: Your model relies on many data sources
#### Problems
:exclamation_mark: If one of the features changes it's distribution - the prediction behavior might change drastically (CACE principle:Changing Anything Changes Everything).

#### Possible solutions
:white_check_mark: **Isolate models based on different sources and serve ensembles.** In some cases this solution may have bad scalability and add cost on maintaining separate models. *Case from my practice: my team used a combination of a linear model and a boosting algorithm to make predictions for out-of-sample objects*.

:white_check_mark: **Gain a deep understanding of your data.** For example, you may build several models on various slices of your data and inspect metrics you receive. *This is an excellent advice in any situation: the better you understand your data and your model - the less surprises you are going to face.*

:white_check_mark: **Add a regularization** punishing for diverging  from  the  prior  model’s  predictions. *This increases chances for your model to achieve the same local minimum it has converged to on the previous run.*

[↩️ Return to the list of symptoms](#symptoms-of-ml-project-technical-debt)

### :question_mark: Your model affects it's own input data
#### Problems
:exclamation_mark: It may make it difficult to analyse system performance and predict its behavior before it is released. In the worst case this feedback loop can be hidden.

#### Possible solutions
:white_check_mark: **Isolate certain parts of data** from the influence of your model.

:white_check_mark: **Identify hidden input loops and get rid of them**. *In general, it requires an understanding of origins of the data you use*.


[↩️ Return to the list of symptoms](#symptoms-of-ml-project-technical-debt)

### :question_mark: Your prediction service has undeclared consumers (aka visibility debt)
#### Problems
:exclamation_mark: Any changes to your model probably will break these silently dependent systems.

:exclamation_mark: It may create a hidden input loop if an undeclared consumer is creating an input data for your model.
#### Possible solutions
:white_check_mark: **Use automated feature management tool** to annotate data sources and build dependency trees. *I believe authors were describing feature store concept before it became widespread*.

:white_check_mark:💡 ***Make your service private** so any consumer within your organization would have to inform you about intentions to use your model output*.

:white_check_mark:💡 ***Support an old version of your prediction service** for some time and make announcements long before any changes*.

[↩️ Return to the list of symptoms](#symptoms-of-ml-project-technical-debt)

### :question_mark: Your model relies on unstable data source (e.g. another model)
#### Problems
:exclamation_mark: Changes in input data source may cause unexpected behavior of your model.
#### Possible solutions
:white_check_mark: **Create a versioned copy of an unstable input data** and use it until the updated version is fully stabilized.

:white_check_mark: **Add more data to teach the first ML-model dealing with your use-case**.

:white_check_mark:💡***Use input features from the first model** to train your own model*.

[↩️ Return to the list of symptoms](#symptoms-of-ml-project-technical-debt)

### :question_mark: You are using features with slim to none performance impact
#### Problems
:exclamation_mark: The more features you have the higher risk that any of them will alter and corrupt your model performance.
#### Possible solutions
:white_check_mark: **Regularly evaluate the effect of removing individual features** from a model.

:white_check_mark: **Develop cultural awareness** about the lasting benefit of underutilized dependency cleanup.

[↩️ Return to the list of symptoms](#symptoms-of-ml-project-technical-debt)

### :question_mark: You have a lot of “glue code” because of a specific ML-package
#### Problems
:exclamation_mark: It may turn into a tax on innovation: switching to other machine learning package would become very expensive.
#### Possible solutions
:white_check_mark: **Re-implement algorithms** from a general-purpose package to satisfy your specific needs. *This may look costly, but sometimes it is the easiest solutions in terms of understanding, testing and maintaining your code. For example my team implemented a common interface for all data transformers and rewrites a code from general-purpose packages like sklearn to suit this interface*.

[↩️ Return to the list of symptoms](#symptoms-of-ml-project-technical-debt)

### :question_mark: Data preparation stages turned into pipeline jungles
#### Problems
:exclamation_mark: Complicated pipelines are difficult to test and maintain.
#### Possible solutions
:white_check_mark: **Do not separate researchers and engineers**, they should work together and probably be one person.

:white_check_mark:💡 ***Use data engineering/MLOps tools** which have become popular nowdays. My team is using Airflow and [DVC](https://dvc.org/) in almost every project which helps us easily manage our data pipelines.*

[↩️ Return to the list of symptoms](#symptoms-of-ml-project-technical-debt)

### :question_mark: You are mixing dead experimental code with a working one
#### Problems
:exclamation_mark: Unused code paths increase system complexity causing a whole range of negative effects from difficulties in maintenance to unexpected behavior of the system.
#### Possible solutions
:white_check_mark: **Build a healthy ML system** which isolates experimental code well. *E.g., [DVC](https://dvc.org/) encourages a usage of separate branches for separate experiments. By doing so, you would nip this problem in the bud*.

[↩️ Return to the list of symptoms](#symptoms-of-ml-project-technical-debt)

### :question_mark: Your configuration files are very complex
#### Problems
:exclamation_mark: Errors in configuration files are a common source of costly mistakes because they are usually not tested properly and treated lightly by engineers.
#### Possible solutions
:white_check_mark: **Validate data passed via configs** using assertions, *e.g. [pydantic](https://pydantic-docs.helpmanual.io/) may help with that*.

:white_check_mark: **Carefully review changes in configuration files**.

[↩️ Return to the list of symptoms](#symptoms-of-ml-project-technical-debt)

### :question_mark: You have chosen a threshold for your model manually
#### Problems
:exclamation_mark: This threshold may become invalid if a model is retrained on new data.
#### Possible solutions
:white_check_mark: Let your ML system to **learn a threshold on holdout data**.

[↩️ Return to the list of symptoms](#symptoms-of-ml-project-technical-debt)

### :question_mark: Your model relies on non-causal correlations
#### Problems
:exclamation_mark: Non-causal correlations may occur randomly or temporarily, so it is extremely risky to rely on them.
#### Possible solutions
:white_check_mark: **Avoid using illogically correlated features**. *Lucky for us, nowadays, a field of causal and explainable ML is developing rapidly.*

:white_check_mark:💡***Check that introduced features affect the results in a way you can explain**, e.g. you may use a combination of domain knowledge and [Shapley Values](https://christophm.github.io/interpretable-ml-book/shapley.html) for that*.


[↩️ Return to the list of symptoms](#symptoms-of-ml-project-technical-debt)

### :question_mark: Your ML-system monitoring and testing require improvements
#### Problems
:exclamation_mark: Unit tests and end-to-end tests are unable to uncover changes in the external world that affect your model behavior.
#### Possible solutions
:white_check_mark: **Monitor prediction bias** *(aka [concept drift](https://en.wikipedia.org/wiki/Concept_drift)).*

:white_check_mark: **Add sanity checks** especially for the systems allowed to perform actions in the real world.

:white_check_mark:💡 ***Monitor other useful metrics**. Here are parameters my team usually monitors in every project: input data distribution, prediction distribution, overall metrics, metrics on some slices of data, feature importance, assertions for edge-cases (e.g. if all features are zero we expect prediction to be zero)*.

[↩️ Return to the list of symptoms](#symptoms-of-ml-project-technical-debt)

## Conclusion
I hope you have found this article helpful! Do not let technical debt cut down an innovation rate of your ML-projects!

If notice an error or just in a mood to say hello, please contact me via [LinkedIn](https://www.linkedin.com/in/vasinkd/), [Telegram](https://t.me/vasinkd) or [email](mailto:vasin.kd@gmail.com). Every message counts, your feedback really motivates me for creation of a new content! Also, you are very welcome to subscribe to my Telegram channel: [@FuriousAI](https://t.me/FuriousAI).