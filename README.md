# Self-Hosting Guide

## Intended Audience

You should be familiar with reading and configuring `docker-compose.yml` files and with infrastructure topics such as DNS and SMTP.

## Pre-Requisites

- A server with 4-8GB RAM, 40GB+ HD, and Docker 20.10.7+ installed.
- Your desired domain (e.g., codecks.example.com) and *.codecks.example.com should be pointing to the server.
- Credentials for email via SMTP or SES (you can also extend the provided docker-compose files by including a service like [Postal](https://docs.postalserver.io/)).

## Introduction

Self-hosting is made possible via Docker images. We'll provide you with user credentials to access these.

To get familiar with the process, we've prepared two sample configurations:

- `sample-local`

  Allows you to set up Codecks on your local machine and access it via `http://localhost:8080`.

- `sample-server`

  Complete server-side setup. Replace all occurrences of `example.com` with your custom domain.

## Setup

- SSH into the target machine.

- Make sure `docker 20.10.7+` is installed.

- Take a look at the provided `docker-compose.yml` and `.env` file, adapt them to your needs, and put them on the target machine.

- Log in to the Codecks Docker Registry and enter the provided credentials:

       docker login dockerhub.codecks.io

- Prepare the database:

       docker compose run db-scripts yarn onprem:initialize

- Set up your account and admin user (all details can be changed within the app afterwards):

      docker compose run \
        -e ORGNAME=MYCOMPANY \
        -e USERNAME="Sarah" \
        -e USEREMAIL="sarah@example.com" \
        -e USERPASSWORD="mypassword" \
        worker node node-api/build/docker/jobs.js

- Start the services:

      docker compose up -d

- If you're using MinIO, create the upload bucket and ensure it's readable by the other services in the network:

  First, find the correct network name via:

      docker network ls

  Look for the one named `[directory_name]_default`.
  Run the following command, replacing `NETWORK_NAME` with the name found above:

      docker run -it --network NETWORK_NAME --env-file .env --entrypoint='' minio/mc:latest sh -c '/usr/bin/mc config host rm local;
      /usr/bin/mc config host add --api s3v4 local ${CUSTOM_S3_ENDPOINT} ${AWS_ACCESS_KEY_ID} ${AWS_SECRET_ACCESS_KEY};
      /usr/bin/mc mb --quiet local/${AWS_UPLOAD_BUCKET}/;
      /usr/bin/mc anonymous set download local/${AWS_UPLOAD_BUCKET};'

- You should now be able to open the page and log in with the provided USEREMAIL and USERPASSWORD above.
- Once logged in, open the Mission Control in the top left to create your first project.

## Update / Migrate

- Modify your `CDX_VERSION` within your `.env` to target the latest version.
- `docker compose down`
- `docker compose run db-scripts yarn onprem:migrate`
- `docker compose up -d`


## Configuration

### Storage for user uploads

Please provide the following ENV variables to the `docker-compose.yml` (e.g., by using an `.env` file):

- `CUSTOM_S3_ENDPOINT` (cdx/node-api needs to be able to access it; depending on your setup, this could be e.g., http://minio:9000).
- `AWS_UPLOAD_HOST` This needs to be an address accessible from the user side. Could be e.g., `https://uploads.codecks.example.com`.
- `AWS_UPLOAD_BUCKET` Pick any name; the bucket will be created below.
- `AWS_ACCESS_KEY_ID` Create your own key here.
- `AWS_SECRET_ACCESS_KEY` Create your own secret here.

### Mail

You can use either SES on AWS or SMTP. You need to provide a `FROM_ADDRESS` variable. This could be a simple email address, but this pattern also works: `"Codecks <mail@codecks.example.com>"`.

### SES on AWS

Please provide the following ENV variables to the `docker-compose.yml` (e.g., by using an `.env` file):

- `AWS_SES_REGION`
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

### SMTP

Please provide the following ENV variables to the `docker-compose.yml` (e.g., by using an `.env` file):

- `SMTP_HOST` (e.g., smtp.gmail.com)
- `SMTP_PORT` (e.g., 587)
- `SMTP_USER` (optional, e.g., example@gmail.com)
- `SMTP_PASSWORD` (optional; e.g., when using Gmail for testing, visit [this page](https://myaccount.google.com/apppasswords) to set up a password, or [this page](https://admin.google.com/ac/apps/gmail/routing) if you're using the Google Suite).

At the moment we only support `authMethod: "plain"`. Please get in touch with us if you require a different setup.

### Codecks-specific variables

- `CDX_VERSION` (e.g., 2.84.0) refers to the latest available version. We'll send you an email once a new update is available.
- `MAX_STORAGE_MBS` available total storage for user uploads in MB. Note that uploads for profile images and deck covers don't count toward this quota. Leave empty to allow unlimited storage.
