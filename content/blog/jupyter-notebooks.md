---
title: "Jupyter Notebooks"
date: 2020-07-05T12:26:39-05:00
draft: false
tags:
  - "python"
  - "data analysis"
  - "jupyter"
---

Here's a collection of some recent data analytics projects I've been working on. If you'd
like to see the notebooks right away, click on the links below; they'll take you to nbviewer,
which renders the notebook for you online. Or clone the repository, create a virtual environment,
and run `pip install -r requirements.txt`.

To see the github repo, click [here](https://github.com/rvlz/data-wrangling).


## Projects
### Cleaning eBay Car Advertisement Data
* [nbviewer](https://nbviewer.jupyter.org/github/rvlz/data-wrangling/blob/master/notebooks/cleaning_ebay_data.ipynb)

### Analyzing Sales Figures Using SQL
* [nbviewer](https://nbviewer.jupyter.org/github/rvlz/data-wrangling/blob/master/notebooks/chinook_sales.ipynb)

### Stack Overflow Data Exploration and Analysis
* [nbviewer](https://nbviewer.jupyter.org/github/rvlz/data-wrangling/blob/master/notebooks/developer_surveys_2017_to_2019.ipynb)


## Installation
Clone projects
~~~
git clone git@github.com:rvlz/data-wrangling.git
~~~

Go to project folder
~~~
cd data-wrangling
~~~

Install virtual environment
~~~
python3 -m venv env
~~~

Activate virtual environment
~~~
. env/bin/activate
~~~

Install project dependencies
~~~
pip install -r requirements.txt
~~~

Launch
~~~
jupyter notebook
~~~
