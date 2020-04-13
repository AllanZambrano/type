---
layout: post
title: "Django Scrapy "
date: 2020-04-13T22:03:28.009Z
tags:
  - django
  - scrapy
---
I hope everyone is staying safe from the corona virus. It has been a really good time to learn new skills and at least try to be somewhat productive.

For this week I have been working with Scrapy and Django. The objective of this post is basically show how to crawl some data and save it on a DB using Django. The whole process of doing this is pretty straight forward: 

* Crawl some data.
* Clean the crawled data.
* Save it on a DB using Django. 
* Show the data.

For this example we will crawl some CoronaVirus Trackers.

First of all we'll need to install and create a new project using scrapy, if you don't know how to install it just follow this [installation guide](http://doc.scrapy.org/en/latest/intro/install.html). There's a pretty cool [tutorial](http://doc.scrapy.org/en/latest/intro/tutorial.html) as well that shows you the basic.

## Extention: scrapy-djangoitem

We will need to install the following extension that allows you to define Scrapy items using existing Django models. More details on their [GitHub](https://github.com/scrapy-plugins/scrapy-djangoitem).

```cmd
pip install scrapy-djangoitem
```

## Django Model

First of all, let's create a new model for the crawled data. 

```python
# models.py
from django.db import models

class Covid(models.Model):
    country = models.CharField(max_lenght=120)
    confirmed = models.IntegerField()
    active = models.IntegerField()
    new_cases = models.IntegerField(default=0)
    deaths = models.IntegerField(default=0)
    new_deaths = models.IntegerField(default=0)
    recovered = models.IntegerField(default=0)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.country
```

And now just add the model to the admin.

```python
#admin.py
from django.contrib import admin
from .models import covid

admin.site.register()
```

## Let's create a Scrapy Project

Before you start scraping, you will have to set up a new Scrapy project. Enter Djangos directory and run:

```shell
scrapy startproject crawler
```

Enter the spiders directory and let's create a new spider. Save the following code in a file named `covid19_spider.py` under the `/spiders` directory in your project:

```python
# covid19_spider.py
import scrapy
from crawler.items import CrawlerCovidItem
from scrapy.loader import ItemLoader


class DataSpider(scrapy.Spider):
    name = "Covid19Crawler"

    custom_settings = {
        'ITEM_PIPELINES': {
            'crawler.pipelines.DataPipeline': 300,

        }
    }

    allowed_domains = ['https://www.worldometers.info']

    start_urls = [
        'https://www.worldometers.info/coronavirus',
    ]

    def parse(self, response):
        for data in response.xpath("//*[@id='main_table_countries_today']/tbody[1]/tr"):
            l = ItemLoader(item=CrawlerCovidItem(), selector=data)
            l.add_xpath('country', './/td[1]')
            l.add_xpath('confirmed', './/td[2]/text()')
            l.add_xpath('active', './/td[7]/text()')
            l.add_xpath('new_cases', './/td[3]/text()')
            l.add_xpath('deaths', './/td[4]/text()')
            l.add_xpath('new_deaths', './/td[5]/text()')
            l.add_xpath('recovered', './/td[6]/text()')

            yield l.load_item()
```

Now we have our crawled data but we need to clean it and attach it to our Django Model. So now lets go to our `items.py`

```python
# items.py
import scrapy
from scrapy_djangoitem import DjangoItem
from scrapy.loader.processors import MapCompose, TakeFirst
from w3lib.html import remove_tags
from CrawledData.models import Covid # Export the model from the django app

def clean_space(param):
    return param.strip(' ')

def clean_number(param):
    return param.strip().replace(',', '')

def clean_plus(param):
    return param.strip().replace('+', '')


class CrawlerCovidItem(DjangoItem):
    django_model = Covid
    country = scrapy.Field(
        input_processor= MapCompose(remove_tags),
        output_processor= TakeFirst()
    )
    confirmed = scrapy.Field(
        input_processor= MapCompose(clean_number),
        output_processor= TakeFirst()
    )
    active = scrapy.Field(
        input_processor= MapCompose(clean_number),
        output_processor= TakeFirst()
    )
    new_cases = scrapy.Field(
        default=0,
        input_processor=MapCompose(clean_space, clean_number, clean_plus),
        output_processor=TakeFirst()
    )
    deaths = scrapy.Field(
        default=0,
        input_processor= MapCompose(clean_space, clean_number, clean_plus),
        output_processor= TakeFirst()
    )
    new_deaths = scrapy.Field(
        default=0,
        input_processor= MapCompose(clean_space, clean_number, clean_plus),
        output_processor= TakeFirst()
    )
    recovered = scrapy.Field(
        default=0,
        input_processor=MapCompose(clean_space, clean_number),
        output_processor=TakeFirst()
    )
```

Now since we have clean our data and attached the Django Model we'll use the pipeline to save our cleaned data on the Database. Head to the `pipeline.py` file and use the following code:

```python
from CrawledData.models import Covid

class DataPipeline(object):
    def process_item(self, item, spider):
        try:
            data = Covid.objects.get(country=item['country'])
            instance = item.save(commit=False)
            instance.pk = data.pk
        except Country.DoesNotExist:
            pass
        item.save()
        return item
```

What this code does is basically avoid duplications. It just updates the country if it's already created. 

Now for a final touch lets create `crawl` command, that will allow us to use `python manage.py crawl` command to crawl our data without the need to access to the crawler directory. 

## Crawl Command