---
slug: adding-sentiment-analysis-to-laravel
issue: adding-sentiment-analysis-to-laravel
title: Are your customers actually happy with your product?
subtitle: Add sentiment analysis to your Laravel app with this one simple trick.
description: Add sentiment analysis to your Laravel app with this one simple trick.
date: 2024-04-10
category: guides
authors:
  - mitchell-davis
published: true
---

I recently spoke at
the [Laravel Sydney meetup](https://www.meetup.com/en-AU/php-laravel-framework-sydney/events/299482326/), where I gave a
demo of how you can add 18 features to your Laravel app using AWS with just a few lines of code.

The talk went great and was received well, and while I wanted to share the recording with you, unfortunately the
recording didn't work properly. To get around this, I've done a writeup of how you can implement sentiment analysis in
your Laravel app, which is a feature that drew a lot of interest from the audience.

## Quick brief on sentiment analysis

Sentiment analysis takes a block of text, generally user-submitted, and runs an algorithm over the top of it to
determine the general sentiment of the text. To put it even more briefly, **we can determine whether the text is
generally positive, negative, neutral**, or mixed (a combination of the other three).

This can be enormously helpful to analyse user-submitted content, and depending on the context or situation, determine
what to do with it.

## Why would you use it?

There are lots of use cases for automation with sentiment analysis; these are just a handful that might apply to your
business or application.

- **Handling customer feedback**

  Let's say you have a widget in your application where customers can send feedback to the business. This might be a
  support widget, or just a general "send us some feedback" widget. In either case, you very likely want to handle
  negative feedback differently from positive or neutral feedback, with an increased response time, or even a different
  escalation process (going right to the manager's inbox, for example).

- **Hiding negative product reviews**

  Let's say you run an e-commerce store, and customers can leave reviews on your product pages. When a new review comes
  in, you might want to analyse it's content and determine if the customer is happy or upset with the product, and
  perhaps even hide a negative review before putting it out there for all to see.

- **Tracking sentiment over time**

  Businesses across all industries want to monitor their image and ensure that they know if the tide is turning against
  them. With sentiment analysis, you can scrape mentions in news articles and social media comments, and plot this on a
  graph to get a general sense of the business' average perceived sentiment over time.

## Interactive demo

Here's an **interactive demo** where you can try it for yourself.

{% demo
demo="sentiment-analysis"
/%}

[AWS charges $0.0001 USD per 100 characters](https://aws.amazon.com/comprehend/pricing/), with a minimum of 300
characters, so every request costs us $0.0003 USD. We have therefore imposed a maximum length of 300 characters, to
avoid excessive charges if this demo gets a lot of traction.

{% call-to-action
title="Looking for help with your Laravel and AWS project?"
/%}

## How can you add sentiment analysis to Laravel?

Easy: **using AWS**.

AWS has a natural language processing service called [Amazon Comprehend](https://aws.amazon.com/comprehend/), and
Comprehend has an API you can use
called [Detect Sentiment](https://docs.aws.amazon.com/comprehend/latest/APIReference/API_DetectSentiment.html).

When you pass it some text and the language that the text is in, it will analyse it and give you back the general
sentiment, plus individual percentages of the confidence that Comprehend's model has of each of the following
sentiments:

- Positive
- Negative
- Neutral
- Mixed

Here's an example request and response:

{% torchlight-collection type="side-by-side" %}

```php {% name="Request.php" %}
$lyrics = implode(PHP_EOL, [
    "We're no strangers to love", // [tl! collapse:start]
    'You know the rules and so do I (do I)',
    "A full commitment's what I'm thinking of",
    "You wouldn't get this from any other guy",
    "I just wanna tell you how I'm feeling",
    'Gotta make you understand',
    'Never gonna give you up',
    'Never gonna let you down',
    'Never gonna run around and desert you',
    'Never gonna make you cry',
    'Never gonna say goodbye',
    'Never gonna tell a lie and hurt you',
    "We've known each other for so long",
    "Your heart's been aching, but you're too shy to say it (say it)",
    "Inside, we both know what's been going on (going on)",
    "We know the game and we're gonna play it",
    "And if you ask me how I'm feeling",
    "Don't tell me you're too blind to see",
    'Never gonna give you up',
    'Never gonna let you down',
    'Never gonna run around and desert you',
    'Never gonna make you cry',
    'Never gonna say goodbye',
    'Never gonna tell a lie and hurt you',
    'Never gonna give you up',
    'Never gonna let you down',
    'Never gonna run around and desert you',
    'Never gonna make you cry',
    'Never gonna say goodbye',
    'Never gonna tell a lie and hurt you',
    "We've known each other for so long",
    "Your heart's been aching, but you're too shy to say it (to say it)",
    "Inside, we both know what's been going on (going on)",
    "We know the game and we're gonna play it",
    "I just wanna tell you how I'm feeling",
    'Gotta make you understand',
    'Never gonna give you up',
    'Never gonna let you down',
    'Never gonna run around and desert you',
    'Never gonna make you cry',
    'Never gonna say goodbye',
    'Never gonna tell a lie and hurt you',
    'Never gonna give you up',
    'Never gonna let you down',
    'Never gonna run around and desert you',
    'Never gonna make you cry',
    'Never gonna say goodbye',
    'Never gonna tell a lie and hurt you',
    'Never gonna give you up',
    'Never gonna let you down',
    'Never gonna run around and desert you',
    'Never gonna make you cry',
    'Never gonna say goodbye',
    'Never gonna tell a lie and hurt you', // [tl! collapse:end]
]);

$request = [
    'LanguageCode' => 'en',
    'Text'         => substr($lyrics, 0, 300),
];
```

```php {% name="Response.php" %}
$response = [
  'Sentiment'      => 'POSITIVE',
  'SentimentScore' => [
    'Positive' => 0.3571358025074005,
    'Negative' => 0.111916184425354,
    'Neutral'  => 0.32517188787460327,
    'Mixed'    => 0.20577609539031982,
  ],
]
```

{% /torchlight-collection %}

From this example, you can see that the lyrics to [Never Gonna Give You Up](https://www.youtube.com/watch?v=dQw4w9WgXcQ)
are overall _positive_ (35.7% confidence), but just barely beating out _neutral_ (32.5% confidence).

The above code shows the general request and response for the call, but here's some actual code we're using in the above
demo to call out to AWS. It assumes you have the [AWS SDK for Laravel](https://github.com/aws/aws-sdk-php-laravel)
installed.

```php
<?php

use Aws\Comprehend\ComprehendClient;

function detectSentiment(string $input, string $language = 'en'): array {
  /** @var ComprehendClient $comprehendClient */
  $comprehendClient = app('aws')->createClient('comprehend');
  
  $result = $comprehendClient->detectSentiment([
    'LanguageCode' => $language,
    'Text'         => $input,
  ]);
  
  return [
    'Sentiment'      => $result->get('Sentiment'),
    'SentimentScore' => $result->get('SentimentScore'),
  ];
}

// ...

dd(detectSentiment('The quick brown fox jumps over the lazy dog')); 
```

To get this to run, you'll need to ensure your IAM credentials are given permission to run `comprehend:DetectSentiment`.

An example policy is below:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": "comprehend:DetectSentiment",
      "Resource": "*"
    }
  ]
}
```

There are plenty more interesting bits of functionality you can add to your Laravel app using AWS, for practically free,
and we'll be writing lots more about them in future articles.

If you enjoyed this demo, **please share it**, and consider subscribing to our newsletter, where we'll announce future
demos and articles on more ways to get the most out of AWS.