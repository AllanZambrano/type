---
layout: post
title: Managing Static Files on Production
tags: [frontpage, django]
---

So today was kinda frustrating, today I learned that Heroku filesystem is ephemeral - that means that any changes to the filesystem whilst the dyno is running only last until that dyno is shut down or restarted. Each dyno boots with a clean copy of the filesystem from the most recent deploy. This is similar to how many container based systems, such as Docker, operate.

This basically means that I can't store images on my Django project cause whenever I push and update it will overwrite and delete the images, heroku recommends using a database addon such as Postgres (for data) or a dedicated file storage service such as AWS S3 (for static files).

It was a very tedious task to make it work and to be honest I think there's something missing, I manage to follow some guides and at the end the images are getting uploaded to S3. 

Source: 

