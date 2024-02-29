---
slug: using-inertia-with-laravel-in-2024
#issue: using-inertia-with-laravel-in-2024
title: Using Inertia with Laravel in 2024
#subtitle: ABC
description: ABC
date: 2024-02-29
category: trends
authors:
  - mitchell-davis
published: false
---

{% callout type="note" title="Mitchell says..." %}
While this article promotes the use of Inertia, I want you to know that [we love Livewire too](/tech-stack). It's a
truly fantastic and exciting technology, with a very bright future, and we mean it no disrespect - I promise 🤝.
{% /callout %}

I was listening to
a [recent episode of the Mostly Technical podcast](https://mostlytechnical.com/episodes/24-whats-the-good-word), where
Ian and Aaron make their case for using Livewire over Inertia. We use Inertia on almost every client project we work on,
so I thought I would help make a case for why you might choose to use Inertia in your Laravel project in 2024.

## A refresher on Inertia

First, let's get up to speed on Inertia.

[Inertia](https://inertiajs.com/) is a protocol created by [Jonathan Reinink](https://twitter.com/reinink), with
supporting libraries for React, Vue, Svelte, PHP, and Rails, with the goal of helping you write single-page apps (SPAs),
without all the complexity that comes with SPAs. Think about managing sessions, tokens, APIs, routing, asset cache
busting. Inertia helps us with all of these by acting as the glue layer between your frontend and backend technologies.

With Inertia applications, you are simultaneously writing both a server-side and a client-side rendered application, and
it all just works seamlessly.

- You write your Laravel routes, controllers, and form requests like you would if you were writing a Blade application.
- You write your React, Vue, or Svelte application like you would if you were writing an SPA, however you can ignore the
  routing component completely as this is handled by Laravel.

For the backend, the main difference is you don't have to write any API code, and in your controllers you
return `Inertia::render('Blog/Index')` instead of `view('blog.index')`.

For the frontend, the main difference is that you get sessions out of the box, so you never have to deal with tokens,
and you don't have to manage your routes, which is a huge benefit.

## How the protocol works

The Inertia docs do a great job of explaining [the protocol](https://inertiajs.com/the-protocol), but the TLDR is that
by calling `Inertia::render()`, after the initial full-page load is made and the Inertia frontend application has
booted, the backend will only ever respond with XHR requests (think JSON payloads, just like an API).

This prevents the server from sending the boilerplate HTML on every pageload, saving you and your users a lot of
bandwidth, and making the application feel very snappy.

Any parameters you pass into `Inertia::render()` will be JSON encoded and sent to the component as props, making them
immediately available to your React, Vue, or Svelte components.

Once you try it, it really feels quite magical to be writing a Vue SPA, with all the benefits Vue gives you, but with
access to your controllers that feels like a Blade application.

Here's an example:

{% torchlight-collection type="side-by-side" %}

```vue {% name="Blog/Index.vue" %}
<template>
  <div class="grid grid-cols-3 gap-6">
    <div v-for="article in $props.articles"> <!-- [tl! focus] -->
      <span class="font-semibold text-lg">{{ article.title }}</span> <!-- [tl! focus] -->
    </div> <!-- [tl! focus] -->
  </div>
</template>

<script>
export default {
  props: { // [tl! focus]
    articles: Array, // [tl! focus]
  }, // [tl! focus]
};
</script>
```

```php {% name="BlogController.php" %}
<?php

namespace App\Http\Controllers;

use App\Models\Article;
use Illuminate\Http\Request;
use Inertia\Inertia;

class BlogController
{
    public function index(Request $request)
    {
        return Inertia::render('Blog/Index', [ // [tl! focus]
            'articles' => Article::query() // [tl! focus]
                ->select([ // [tl! focus]
                    'id', // [tl! focus]
                    'title', // [tl! focus]
                ]) // [tl! focus]
                ->latest() // [tl! focus]
                ->limit(9) // [tl! focus]
                ->get(), // [tl! focus]
        ]);
    }
}

```

{% /torchlight-collection %}

This is some spacer text 😜

{% torchlight-collection %}

```php {% name="BlogController.php" %}
<?php

namespace App\Http\Controllers;

use App\Models\Article;
use Illuminate\Http\Request;
use Inertia\Inertia;

class BlogController
{
    public function index(Request $request)
    {
        return Inertia::render('Blog/Index', [
            'articles' => Article::query()
                ->select([
                    'id',
                    'title',
                ])
                ->latest()
                ->limit(9)
                ->get(),
        ]);
    }
}

```

```vue {% name="Blog/Index.vue" %}
<template>
  <div class="grid grid-cols-3 gap-6">
    <div v-for="article in $props.articles">
      <span class="font-semibold text-lg">{{ article.title }}</span>
    </div>
  </div>
</template>

<script>
export default {
  props: {
    articles: Array,
  },
};
</script>
```

{% /torchlight-collection %}

## The perks of using Inertia with Laravel

## The negatives of using Inertia with Laravel

### You can't leverage Blade

_I'm not a huge fan of Blade_. There, I said it.

It's an amazing layer on top of vanilla PHP syntax, and is great as a templating language, but it's always felt
cumbersome to use.

- I don't like that you don't know which PHP variables you have access to inside your Blade file, without checking the
  controller (or _controllers_) that rendered your Blade view.
- I don't like that you can make database calls from your presentation layer. Given that it compiles down to just PHP,
  Blade doesn't force you to think about the data you're making available to your views.
- I don't like the lack of good IDE support.

If you _are_ a huge fan of Blade, you lose access to everything you love about it by using Inertia.

It really is a case of choosing one or the other.

### You might not need an SPA

You'll have to decide for yourself if you really need all the power that comes from a full SPA, at the cost (for those
that see it that way) of writing your whole frontend in JavaScript.

We have started many projects where we didn't think we would need complex JS, only to find out later that it would be
really handy to be able to use some feature of [Headless UI](https://headlessui.com/), and to have to shoehorn it in, or
build it ourselves with [Alpine.js](https://alpinejs.dev/).

### Maintenance, new features, and "complete software"

During the podcast, Aaron brought up the infamous modals presentation, which was a new and exciting feature
that [Jonathan previewed](https://youtu.be/euKphNYApn4?t=1550) during Laracon Online 2021. It appeared to show a way to
manage modals over the top of regular pages directly from your controller, giving you a direct route to create a new
post (or whatever your domain model is), while still displaying the list of posts in the background underneath it.

```yaml {% theme="github-light" %}
name: EAS Build
on:
  workflow_dispatch:
    inputs:
      platform:
        description: 'Which platform to build for?'
        required: true
        type: choice
        default: all
        options:
          - ios
          - android
          - all
  push:
    branches:
      - production
jobs:
  build:
    name: Install and build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18.x
          cache: npm
      - name: Setup Expo and EAS
        uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      - name: Install dependencies
        run: npm ci
      - name: Set the variables
        run: echo "DEPLOY_PLATFORM=${{ inputs.platform || 'all' }}" >> $GITHUB_ENV
      - name: Build on EAS and Auto-Submit to App/Play Stores if successful
        run: eas build --platform "$DEPLOY_PLATFORM" --non-interactive --no-wait --profile production --auto-submit
```


This feature was never released, which led me at least to some feelings of abandonment. Jonathan wrote about why it got
cut from the scope for v1.0 on [Reddit](https://www.reddit.com/r/laravel/comments/10bsc2o/comment/j4bul56/). Given
Jonathan has been working for the folks at [Tailwind Labs](https://tailwindcss.com/) for the last few years, **naturally
and completely understandably** Inertia has appeared to become less of a focus for him personally.

Fortunately for all of us, [Taylor Otwell](https://twitter.com/taylorotwell), the founder and CEO of Laravel, has
dedicated some of the Laravel core team to maintaining the Inertia repositories, so we are in very good hands there.

I completely agree with Ian and Aaron that a big drawback of Inertia is that it is "_complete software_", meaning it's
not expected to get any further features. Despite it getting regular upgrades for dependencies and such, this does make
it _feel_ abandoned. Taylor tweeted about this
in [July 2023](https://twitter.com/taylorotwell/status/1684999340446900224).

{% tweet url="https://twitter.com/taylorotwell/status/1684999340446900224" /%}

## How to make the most of Inertia

- Use Ziggy
- Use forms
- Use middleware to share key data

_Begin by setting the stage for the current web development landscape, emphasizing the continuous evolution of
technologies and the need for solutions that balance efficiency, performance, and developer experience. Introduce
Inertia.js as a bridge between traditional server-driven applications and modern single-page applications (SPAs),
specifically within the context of Laravel projects._

## The Evolution of Web Development

_Provide a brief overview of how web development practices have evolved, from multi-page applications (MPAs) to SPAs.
Discuss the rise of API-driven development and the complexities it introduced. This section sets the historical context
for why Inertia.js is relevant today._

## Understanding Inertia.js

Explain what Inertia.js is and how it works, focusing on its role as a bridge that allows developers to create SPAs
without a separate API. Highlight its key features, such as tight backend integration, the use of existing server-side
routing and controllers, and seamless page transitions.

## Why Inertia.js with Laravel?

Delve into the specific advantages of using Inertia.js with Laravel, such as:

- Simplified development workflow by leveraging Laravel's robust backend capabilities alongside Vue, React, or Svelte
  for the frontend.
- Reduced boilerplate code and complexity compared to traditional SPA development, which often requires an API.
- Enhanced user experience through faster page loads and app-like interactions, without sacrificing SEO and server-side
  rendering benefits.
- Strong community support and a growing ecosystem of tools and resources.

## Respectful Comparison with Other Technologies

Without diminishing other technologies, present a comparison that highlights how Inertia.js offers a unique approach.
For example, compare it to:

- Traditional SPA frameworks (like Angular, React, or Vue used with separate backends), emphasizing Inertia's
  elimination of the need for an API layer.
- Server-side rendering (SSR) solutions, discussing how Inertia.js provides an alternative that maintains the benefits
  of SSR while offering the interactive advantages of SPAs.
- Mention the scenarios where traditional SPAs or SSR might still be the preferred choice, such as applications
  requiring highly decoupled frontends and backends or those heavily reliant on microservices.

## Case Studies and Success Stories

Share examples from your own experience or the broader community where Inertia.js with Laravel has led to successful
outcomes. Highlight specific challenges it helped overcome, performance improvements, or developer productivity gains.

## Getting Started with Inertia.js and Laravel

Offer practical advice for developers looking to start with Inertia.js in their Laravel projects. Include tips on setup,
learning resources, and best practices for structuring projects for scalability and maintainability.

## The Future of Web Development with Inertia.js and Laravel

Conclude by reflecting on the potential of Inertia.js and Laravel to shape the future of web development. Encourage
readers to explore this approach as a way to build efficient, user-friendly applications while staying aligned with
modern development trends.

## Call to Action

Invite readers to share their experiences, questions, or insights on using Inertia.js with Laravel in the comments
section. Encourage them to explore further and consider how this approach might benefit their projects.