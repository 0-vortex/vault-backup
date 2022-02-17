---
canonicalUrl: https://dev.to/opensauced/deploy-a-github-application-to-cloudflare-workers-ie5-temp-slug-4228941?preview=b3d945ce619e56bd0b645a2c92aa84316538cbdb0b3be0b9971204dd672d448724e2d9732115aa955053002e3b23cd9f2308164a06028dbb0889a3e7
tags:
- GitHub actions
- Octokit
- nodejs
---

# Deploy a GitHub Application to Cloudflare Workers

![[black-and-white-bridge-fog-infinite.jpg]]

## Introduction 

> "why, why, WHY?"

After seeing [@bdougieyo](https://dev.to/bdougieyo) build a [ProBot](https://bdougie.github.io/sls-probot-guide/) app and [@blackgirlbytes](https://dev.to/blackgirlbytes) fresh take on [deploying ProBot to AWS Lambda](https://dev.to/github/deploying-my-probot-app-to-a-serverless-lambda-352h), I figured I would spice things up a bit by researching the most financially cost-effective solution to run a serverless GitHub application on.

Now hold on, you might be thinking things like:
- only care about money?!
- AWS Lambda is dirt cheap!
- it's all a configuration war you cannot win!

![[Screenshot 2022-02-17 at 00.56.59.png]]

Me: "Hold up, whaaaat?!"
Other me: "Yes it's [CloudFlare Workers](https://workers.cloudflare.com)!"

![](https://media.giphy.com/media/21I1WOUDnct4EmSNa6/giphy.gif)

The simple explanation is it's using the Service Worker API, they offer a flat, free, 100.000 requests a day if you [can keep it cutting edge](https://developers.cloudflare.com/workers/platform/limits#worker-limits), has local development and testing options with [miniflare](https://miniflare.dev) and a [KV store](https://www.cloudflare.com/en-gb/products/workers-kv/).

Still sounds fishy? That's because by default it's using Webpack 4, so it can do Rollup, so it can do Vite. Yes, [@mtfoley](https://dev.to/mtfoley), this is preparing to be another [converting to Vite series](https://dev.to/mtfoley/series/16147)!

## Technical part

> "how to how-to X!"

### Requirements

This is going to hurt:
- make the existing Probot code compatible
- writing less than browser compatible code
- <10 ms CPU execution time due to the workers' limits
- automated releases over an open-source repository 
- secure deployments

### Code

Assuming [the workers PR](https://github.com/open-sauced/catsup-app/pull/2) will eventually be production ready, the code should be visible over at:

{% github open-sauced/catsup-app %}

In order for the project to be shipped as a service function, the node environment can not be used in any of the production code. While this seems like a dead-end for Probot - I'm looking at you [`require("dotenv").config()`](https://github.com/probot/probot/blob/5d232e2d86c72cff541d193a877a4ccf90cea6d7/src/bin/probot.ts#L5) - its underlying framework, [OctoKit](https://github.com/octokit/octokit.js) does not come with any opinionated code.

Simply expanding the script into the Probot equivalent while dodging the node imports was very easy and done before, being able to see existing working code made the process a lot more enjoyable:

{% github gr2m/cloudflare-worker-github-app-example %}

Using the same [probot/smee-client](https://github.com/probot/smee-client) shipped by Probot we divert the webhook URL to localhost for the development application - for the production application we will enter a custom route.

While it might look like an anti-pattern, setting up a private local-only application in the workers configuration file is perfectly safe and meant as the most basic way to ensure all environment variables are encrypted for the production environment. A neat trick to environment variables in workers, we are unable to deploy the app if the variables don't exist and the only way to add them is by encrypting them as secrets.

The above secret pattern requires us we set up the GitHub application and Discord hooks before trying to deploy the service worker, as it would otherwise fail with unencrypted or loose values.

### Setting up the service worker

#### 1. Cloudflare worker

Set up a cloudflare account and enable workers, change `account_id` in [wrangler.toml](./wrangler.toml) to your account id.  
  
Go to your workers dashboard and create a new worker, select any template, adjust `name` in [wrangler.toml](./wrangler.toml) if the existing one is taken.

Write the "Routes" URL provided by the worker down somewhere for the next parts. It will serve as webhook return URL.

#### 2. GitHub application

Create a new GitHub application with scopes `issues:write` and `metadata:read` while also enabling tracking events.  
  
Upon creation you should have plain-text values for `APP_ID`, `CLIENT_ID`.  
  
Click the "Generate a new client secret" button and copy the value of `CLIENT_SECRET`.  
  
In the webhook return URL copw the value of your worker route as described in the last step of the Cloudflare setup.  
  
It is advised you generate the `WEBHOOK_SECRET` using the following command:

```shell  
# random key strokes can work too if you don't have ruby(??)  
ruby -rsecurerandom -e 'puts SecureRandom.hex(20)'  
```  
  
Now, go to the very bottom and click "Generate a new private key" and open a terminal in the location of the downloaded file.  
  
Rename this file to `private-key.pem` for the next command to work:  
  
```shell  
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in private-key.pem -out private-key-pkcs8.key
```  
  
Copy the contents of `private-key-pkcs8.key` to `APP_PK`.

#### 3. Discord webhook

Go to your server of choice, click "Settings" and then "Integrations", create a new webhook and copy the URL and paste that value into `DISCORD_URL`.  
  
Now you are good to use the wrangler release workflows and deploy to production!

#### 4. Environment variables

Select the "Settings" tab on your newly created worker and click "Variables", add the following variables with the values described in the previous steps:

-   `APP_ID`
-   `APP_PK`
-   `DISCORD_URL`
-   `CLIENT_ID`
-   `CLIENT_SECRET`
-   `WEBHOOK_SECRET`

Encrypt all of them and deployment will start working both locally and in the CI workflows! 

### Deployment

The PR code, as well as the maintainers, aro not yet sure what the best wey to approach multiple environment deployment should look like. There is a small consideration for the CI action leaking the target URL which would give the possibility of service disruption - making the deployment target fully private, IE deploying from `wrangler` locally would make the discovery process partially visible to us in application installations and limit outbound attack vectors considerably. Sitting behind 2 of the biggest global CDNs also help a lot!

#### Local publish

Login to cloudflare with your account credentials, advised you let the browser open an OAuth dialog with:  
  
```shell  
npm run wrangler -- login
```  
  
Now you can test all the variables are correct by publishing from the terminal:  
  
```shell  
# npm run wrangler -- publish  
npm run publish  
```  
  
Open up a production real time log using:  
  
```shell  
npm run wrangler -- tail
```

#### GitHub actions

Create a new GitHub actions secrets named `CF_API_TOKEN`, get its value from Cloudflare's [create a new token](https://dash.cloudflare.com/profile/api-tokens) using the "Edit Cloudflare Workers" template.  
  
Push new code to the server, after a release the new code should be sent to the server and instantly propagate.

## Conclusion

> "easy, what next?!"

Some things come to mind as definitely needed:
- setting up miniflare for better local development
- verifying the signature and returning 404/500 on error
- switching the build system to rollup/vite
- implement testing and coverage
- move secrets to KV namespaces for easier environment deployments
- dockerize repository
