---
slug: using-inertia-with-laravel-in-2024
issue: using-inertia-with-laravel-in-2024
title: Using Inertia with Laravel in 2024
description: We are all-in on Inertia in 2024. Here's why you might want to be too.
date: 2024-03-05
category: trends
authors:
  - mitchell-davis
published: false
---

I was listening to
a [recent episode of the Mostly Technical podcast](https://mostlytechnical.com/episodes/24-whats-the-good-word), where
Ian and Aaron make their case for using Livewire over Inertia. We use Inertia on almost every client project we work on,
so I thought I would help make a case for why you might choose to use Inertia in your Laravel project in 2024.

{% callout type="note" title="Livewire is great too!" %}
While this article promotes the use of Inertia, I want you to know that [we love Livewire too](/tech-stack). It's a
truly fantastic and exciting technology, with a very bright future, and we mean it no disrespect - I promise 🤝.
{% /callout %}

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

For the frontend, the main difference is that you write your frontend in Vue, React, or Svelte; you get sessions out of
the box, so you never have to deal with tokens, and you don't have to manage your routes, which is a huge benefit.

### How the protocol works

The [Inertia docs](https://inertiajs.com/the-protocol) do a great job of explaining the protocol, but the TLDR is that
by calling `Inertia::render()`, after the initial full-page load is made and the Inertia frontend application has
booted, the backend will only ever respond with XHR requests (think JSON payloads, just like an API).

This prevents the server from sending the boilerplate HTML on every pageload, saving you and your users a lot of
bandwidth, and making the application feel very snappy.

Any parameters you pass into `Inertia::render()` will be JSON encoded and sent to the component as props, making them
immediately available to your React, Vue, or Svelte components. Your frontend application will interpret the response
from the backend, and render the appropriate component, using the props you passed to it.

Once you try it, it really feels quite magical to be writing a Vue (or others) SPA, with all the benefits Vue gives you,
but with
access to your controllers that feels like a Blade application.

Here's an example:

{% torchlight-collection type="side-by-side" %}

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
        ]); // [tl! focus]
    }
}
```

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

{% /torchlight-collection %}

## Why you should use Inertia with Laravel

### You don't have to create an API

Did you ever write an Angular frontend back in the day, and had to build out a separate API for your frontend to
call? **We did.**

Did you have to figure out where to store your authentication token in the browser? **We did.**

Did you have to manage sessions expiring? **We did.**

Inertia removes all of these problems and more, because you aren't writing an API - you're writing a typical Laravel
application that uses JavaScript instead of Blade.

- You get sessions without needing authentication tokens.
- You get form requests and redirects without needing API endpoints.

Sure, you _can_ write actual API routes if you need them, and call them from your Inertia app, but it's not a
requirement. And if you do, Inertia will use your session to make those calls, so you're covered there as well!

### Your app will be really fast

You get a big performance boost by using Inertia, mostly because the backend only has to send through the actual data
(as JSON props) needed to render your component, and not the HTML for those components.

Because the components themselves are just JavaScript files, they will be cached by the browser and by your CDN, so
until you release an update to your application, you literally just send the user the HTML for your component once.

This differs from a Blade application, where you are sending the HTML with every single request and are unable to cache
it in the browser, assuming there is some dynamic content on the page.

### All JavaScript, all of the time

With Inertia, there is almost no
Blade (technically [two lines](https://inertiajs.com/server-side-setup#root-template) of Blade are required), so you
spend all of your frontend time inside JavaScript, never having to make a conscious decision to implement a bit of
functionality in Blade or to bring in some JS sprinkles.

In addition, putting your JS inside your Blade files has just never felt right to me. Now you can have your IDE support
for Vue, React, or Svelte, as you are **actually** writing a Vue, React, or Svelte component, with the right file
extension.

### You can still do server-side rendering

Inertia has [great support](https://inertiajs.com/server-side-rendering) for server-side rendering, which will help keep
your initial load really quick, and makes it easier for search engines to index your page.

It's really easy to setup using the packages that Inertia makes available.

## Why you might not use Inertia with Laravel

### You can't leverage Blade

_I'm not a huge fan of Blade_. There, I said it.

It's an amazing layer on top of vanilla PHP syntax, and is great as a templating language, but it's always felt
cumbersome to use.

- I don't like that you don't know which PHP variables you have access to inside your Blade file, without checking the
  controller (or _controllers_) that rendered your Blade view.
- I don't like that you can make database calls from your presentation layer. Given that it compiles down to just PHP,
  Blade doesn't force you to think about the data you're making available to your views.
- I don't like the lack of good IDE support.

If you _are_ a huge fan of Blade, you lose access to everything you love about it by using Inertia. It really is a case
of choosing one or the other.

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

## How to get the most from Inertia

If you're keen to get started with Inertia, here are our top tips for learning to love it.

### Use Ziggy for routing

Love using the `route()` helper in Blade? If you're not using it already,
you [definitely should](https://laravel.com/docs/master/helpers#method-route).

You can _effectively_ achieve the same result using Tigten's [Ziggy](https://github.com/tighten/ziggy) package.
Ziggy will take your Laravel routes, serialize them as JSON, and embed them in your HTML when the page first loads.
Route parameters, domains, prefixes, etc. will all be included too.

Ziggy will give you a `route()` method on the `window` object, which you can use to dynamically create the route string.
Ziggy will also give you a `route().current()` method, which you can use as a truth test to see if the route you
pass in is the route that is currently being visited. This is super helpful for creating dynamic navbars.

Here's an example comparing Blade and Vue.

{% torchlight-collection type="side-by-side" %}

```blade {% name="posts/index.blade.php" %}
<div class="grid grid-cols-3 gap-6">
  @foreach($posts as $post)
  <div>
    <a href="{{ route('posts.show', ['post' => $post]) }}" 
       class="font-semibold text-lg"
    >{{ $post->title }}</a>
  </div>
  @endforeach
</div>
```

```vue {% name="Posts/Index.vue" %}

<template>
  <div class="grid grid-cols-3 gap-6">
    <div v-for="post in $props.posts">
      <Link :href="route('posts.show', {post: post.id})"
            class="font-semibold text-lg"
      >{{ post.title }}
      </Link>
    </div>
  </div>
</template>

<script setup>
import {Link} from '@inertiajs/vue3';
</script>

<script>
export default {
  props: {
    posts: Array,
  },
};
</script>
```

{% /torchlight-collection %}

### Use Inertia forms and Laravel form requests

ABC.

### Use Inertia middleware to share common data across pages

ABC.

### Use Inertia events for triggering analytics

You can hook into all of the Inertia events using the `router`, which makes it trivial to track pageviews as users
navigate your application.

{% torchlight-collection type="tabs" %}

```js {% name="app.js" %}
import './bootstrap'; // [tl! collapse:start]
import '../css/app.css';
import {
	createApp,
	h,
} from 'vue';
import {
	createInertiaApp,
	Head as InertiaHead,
	Link as InertiaLink,
	router,
} from '@inertiajs/vue3'; // [tl! collapse:end]

createInertiaApp({
	resolve: (name) => resolvePageComponent( // [tl! collapse:start]
		`./Pages/${name}.vue`, import.meta.glob([
			'./**/*.vue',
			'../img/**',
		])), // [tl! collapse:end]

	setup({
		      el,
		      App,
		      props,
		      plugin
	      }) {
		const vueApp = createApp({ // [tl! collapse:start]
			methods: {
				trackFathomPageview: function () {
					if (!this.$inertia.page.props.meta) {
						return;
					}

					let url = this.$inertia.page.props.meta.fathom.canonical_url || '';

					if (window.fathom && url) {
						fathom.trackPageview({
							url: url,
						});
					}
				},
			},

			mounted() {
				setTimeout(() => {
					this.trackFathomPageview();
				}, 100);
			},

			render: () => h(App, props),
		})
			.mixin({methods: {route}}) // Defined in the Ziggy package
			.component('InertiaHead', InertiaHead)
			.component('InertiaLink', InertiaLink)
			.use(plugin);

		const mountedVueApp = vueApp.mount(el); // [tl! collapse:end]

		router.on('navigate', (event) => { // [tl! **]
			mountedVueApp.trackFathomPageview(); // [tl! **]
		}); // [tl! **]
	},
});
```

{% /torchlight-collection %}

## Final thoughts

Hopefully it's clear by now that we love Inertia, and have no plans to move away from it any time soon. We are satisfied
with the fact that Inertia is very unlikely to get new features into the future, and grateful for the ongoing
maintenance being completed by the Laravel core team.

If you're looking for further reading, we encourage you to check out the
official [Inertia documentation](https://inertiajs.com/), and to also take a look at
the [Advanced Inertia](https://advanced-inertia.com/) course by Boris Lepikhin.

{% call-to-action
title="Looking for help with your Inertia project?"
buttonText="Contact us today"
/%}