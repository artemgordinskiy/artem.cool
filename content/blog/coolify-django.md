+++
draft = false 
date = 2020-11-22T15:06:12+01:00 
title = "Migrating from Kamal to Coolify"
description = "My Experience"
slug = "coolify-django"
authors = ["Artem Gordinskiy"]
tags = [] 
categories = []
externalLink = ""
series = ["Blog"]
+++

Today, I migrated two of my Django ([Pegasus](https://www.saaspegasus.com/?via=artem)) apps from Kamal to [Coolify](https://coolify.io/), and I wanted to tell you how it went. If you're considering a similar move, read on for a breakdown of what to expect and let me know if you have any questions!

## Setting up Coolify

The setup process was pretty smooth. I spun up a fresh VPS on [Hetzner](https://hetzner.cloud/?ref=yGcKl9KVNC1w) and followed [this excellent guide from CJ@Syntax](https://www.youtube.com/watch?v=taJlPG82Ucw) to get everything up and running.

## Deployment

I opted for a [Docker Compose-based deployment](https://coolify.io/docs/knowledge-base/docker/compose/), which seems to work well for deploying a monolithic Django app with Celery (+ Celery Beat).

### Steps to Deploy an App with PostgreSQL and Redis

Here’s how I deployed my app with PostgreSQL and Redis, including some gotchas I spent hours figuring out:

1. **Create a New Project**: Start by creating a new "Project" in the Coolify UI.
2. **Add Resources**:
    - Add a PostgreSQL resource. This can be done easily in the UI. [Here’s the guide](https://coolify.io/docs/databases/).
    - Do the same for Redis.
3. **Add Your App Resource**:
    - Choose "Docker Compose" as the build option after linking your app repository.
    - Here’s an example `docker-compose.yml` file I made (add it to your repository beforehand):

    ```yaml
    version: '3.8'

    services:
      my_app_web:
        build:
          context: .
          dockerfile: Dockerfile.web
        environment:
          PORT: ${PORT}
          DJANGO_SETTINGS_MODULE: my_app.settings_production
          POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
          SECRET_KEY: ${SECRET_KEY}
          DATABASE_URL: ${DATABASE_URL}
          REDIS_URL: ${REDIS_URL}
        restart: unless-stopped
        ports:
          - "${PORT}:8000"

      celery:
        build:
          context: .
          dockerfile: Dockerfile.web
        command: celery -A my_app worker -l INFO --concurrency 4
        environment:
          DJANGO_SETTINGS_MODULE: my_app.settings_production
          POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
          SECRET_KEY: ${SECRET_KEY}
          DATABASE_URL: ${DATABASE_URL}
          REDIS_URL: ${REDIS_URL}

      celery_beat:
        build:
          context: .
          dockerfile: Dockerfile.web
        command: celery -A my_app beat -l INFO
        environment:
          DJANGO_SETTINGS_MODULE: my_app.settings_production
          POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
          SECRET_KEY: ${SECRET_KEY}
          DATABASE_URL: ${DATABASE_URL}
          REDIS_URL: ${REDIS_URL}
        restart: unless-stopped
    ```
4. **Environment Variables**: Add all necessary environment variables in the resource settings. You can copy the connection URLs for PostgreSQL and Redis from the UI after creating those resources.
5. **Connect to Predefined Network**: Enable this option for your app resource to connect it to the same Docker network as your PostgreSQL and Redis resources.
6. **Domain Setup**: Assign a domain for your Django app, including your `$PORT`. For example, `https://app.coolstuff.com:8000`. N.B. Port 8000 is actually taken by Coolify itself.
7. **Deploy**: Hit deploy and check the logs to see if everything worked 🤞

## Pros and Cons (So Far)

### Pros
- **Continuous Deployment**: Push to deploy. Simply pushing your changes to GitHub to deploy is a much nicer experience than running commands and burning through your battery by building images with Kamal.
- **Notifications**: Easy to set up Telegram/email notifications for deployment events.
- **Ease of Use**: Deploying new apps to your server is straightforward compared to Kamal.
- **Active Development**: Coolify has an active development community and is evolving quickly.
- **Cost-Effective**: It’s much cheaper than using a commercial PaaS.

### Cons
- **Beta Version**: The current major version of Coolify is still in beta so it has some rough edges.
- **Manual Setup**: You can't set up your entire stack with a single config file — you need to click around the UI quite a bit.
- **UI Performance**: The UI can be a bit slow, especially on smaller instances. Hopefully, this is just a "beta thing" and the backend gets optimised soon.
- **Port Conflicts**: Port 8000 is reserved by Coolify itself. And since all the apps are running on the same host, different apps/resources can’t use the same port.

That’s it for now! All in all, this was a fun experience, despite the fact that I hit a bug during installation and had to restart the process, and there were a few other hiccups* along the way. 

_*It would have probably been a smoother process had I been more patient and actually read the documentation but my nature is to [jump in first](https://www.youtube.com/watch?v=mLyOj_QD4a4) and learn to swim later :D_
