---
layout: post
title: Serving static & media files on Heroku
tags: [frontpage, django]
comments: true
---
The whole problem that I got while deploying my app, as mentioned before, was serving both static and media files, whenever I upload a new version of the code my images are deleted since they are save on the project folder.

I managed to find a a pretty good guide over the internet that resolves this issues on Heroku.
Step 1: Serving static assets

Django only serve media files and static assets in dev mode, so I will show you how to serve them on Heroku in production mode.

To serve static assets, we need to use a 3-party package. whitenoise.
```bash
$ pipenv install whitenoise
```
Edit MIDDLEWARE in settings.py, put it above all other middleware apart from Django’s SecurityMiddleware:
```python
MIDDLEWARE = [
   'django.middleware.security.SecurityMiddleware',
   'whitenoise.middleware.WhiteNoiseMiddleware',
   # ...
]
```
Set in `settings.py`
```python
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage' 
```
After roll a `python manage.py collectstatic --noinput --clear`, all static assets would be put to static_root directory.

Step 2: Serving media files
Because all changes to Heroku would be lost during redploy. So we need to store our media files to some other place instead of Heroku.

The popular solution is to use Amazon S3 storage, because the service is very stable and easy to use.
1. If you have no Amazon service account, please go to Amazon S3 and click the Get started with Amazon S3 to signup.
2. Login AWS Management Console
3. In the top right, click your company name and then click My Security Credentials
4. Click the Access Keys section
5. Create New Access Key, please copy the AMAZON_S3_KEY and AMAZON_S3_SECRET to notebook.

Next, we start to create Amazon bucket on S3 Management Console, please copy Bucket name.

Now let's config Django project to let it use Amazon s3 on Heroku.
```bash
$ pipenv install boto3
$ pipenv install django-storages
```
Add storages to `INSTALLED_APPS` in settings.py

Add config below to `settings.py`
```python
AWS_STORAGE_BUCKET_NAME = env('AWS_STORAGE_BUCKET_NAME')
AWS_S3_CUSTOM_DOMAIN = '%s.s3.amazonaws.com' % AWS_STORAGE_BUCKET_NAME
AWS_ACCESS_KEY_ID = env('AWS_ACCESS_KEY_ID')
AWS_SECRET_ACCESS_KEY = env('AWS_SECRET_ACCESS_KEY')
AWS_S3_FILE_OVERWRITE = False

MEDIA_URL = "https://%s/" % AWS_S3_CUSTOM_DOMAIN
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
```

1. To secure your Django project, please set AWS_STORAGE_BUCKET_NAME, AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY in env instead of project source code.
2. AWS_S3_FILE_OVERWRITE plesae set it to False, so this can let the storage handle duplicate filenames problem.

Source: [Deploy Django Project Heroku Using Docker](https://www.accordbox.com/blog/deploy-django-project-heroku-using-docker/)