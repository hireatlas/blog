---
slug: laravel-pipelines-for-managing-complexity
issue: laravel-pipelines-for-managing-complexity
title: "Laravel Pipelines - Wrangling complexity in a maintainable way"
description: "How we use Laravel Pipelines to take control of complex processes"
date: 2024-04-18
category: guides
authors:
  - chris-rossi
image: /assets/articles/laravel-pipelines-for-managing-complexity.png
published: false
---

Pipelines are one of Laravel's more overlooked features, but arguably something every
Laravel developer should keep in their tool belt.

The [official Laravel Documentation](https://laravel.com/docs/11.x/helpers#pipeline) only includes a brief section on this feature but this is more than enough to get started.
Pipelines are actually used under the hood by Laravel [Middleware](https://laravel.com/docs/11.x/middleware), so if you have written your own middleware you should already be familiar with the way Pipelines work.

In this article, we show how we used this powerful feature to wrangle complex business processes and make our codebase significantly easier to maintain.

## Getting started with Pipelines

The Laravel `Pipeline` facade allows you to pass a value through multiple "pipes", which can then read/modify the value as needed at each step.
Each step in the Pipeline passes its output into the next step, allowing complex logic to be broken down into simple and maintainable classes. 

As an example, imagine we have an employee `Timesheet` model, which requires a series of automated checks upon completion.
If any checks fail, we need them to be recorded against the `flags` attribute so that it can be reviewed manually later.

We could have the following checks defined as individual classes:
1. `LongDurationsCheck` - Flag if the total duration exceeds a predefined value (eg 12hrs) 
2. `MandatoryReviewCheck` - Flag if the employer has enforced manual reviews of all Timesheets
3. `AutomaticallyFinishedCheck` - Flag if the employee forgot to sign out after their shift and it was automatically finished by the system

... and so on.

Our pipeline could look something like this:
```php
app(Pipeline::class)
    ->send($timesheet)
    ->through([
        LongDurationsCheck::class,
        MandatoryReviewCheck::class,
        AutomaticallyFinishedCheck::class,
    ])
    ->via('handle')
    ->then(function (Timesheet $timesheet) {
        // Save the changes
        $timesheet->save();
        
        // TODO: We could dispatch a notification/email here to alert the employer
        // that there is a Timesheet requiring review
        }
    });
```

This could be called from a Controller, or (more likely) a queued job dispatched by the controller upon Timesheet completion.

The Pipe for checking for long durations could look like this:
```php
class LongDurationsCheck
{
    // 12hrs in seconds
    const MAX_DURATION_SECONDS = 43200;
    
    public function handle(Timesheet $timesheet, Closure $next): void
    {
        if ($timesheet->duration >= self::MAX_DURATION_SECONDS) {
            $timesheet->addFlag('long-duration');
        }

        $next($timesheet);
    }
}
```

The Pipe for mandatory reviews should only add the flag if the Employer has this feature enabled:

```php
class MandatoryReviewCheck
{
    public function handle(Timesheet $timesheet, Closure $next): void
    {
        if ($timesheet->employer->requires_manual_review) {
            // This employer requires all timesheets to be manually reviewed
            $timesheet->addFlag('manual-review');
        }

        $next($timesheet);
    }
}
```

## Using Pipelines for Advanced Filtering

In one of our more complex scenarios, we need to return a filtered list of matching available Candidates when posting a new job.

This requires more than 10 different checks to be performed in order to determine if each Candidate is eligible for that specific job.

To make this process more manageable, we use a custom `FilterContext` class to hold all the required data for the filtering.

```php
class FilterContext
{
    private Collection $candidates;
    
    public function __construct(
        private Job $job,
    ) {
        $this->candidates = collect();        
    }
    
    public function getCandidates(): Collection
    {
        return $this->candidates;
    }
    
    public function setCandidates(Collection $candidates): self
    {
        $this->candidates = $candidates;
        return $this;
    } 
    
    public function getJob(): Job
    {
        return $this->job;
    }
}
```

We then pass this through our Pipeline:
```php
// Create a job with a start/finish time and a Location, and several mandatory skills
$job = Job::create([
    'title' => 'Lead Forklift Driver',
    'start' => '2024-05-01 09:00:00',
    'finish' => '2024-05-01 17:00:00',
    'location_id' => 12345,
    // First Aid Training and Forklift Driving are mandatory for this job
    'skills' => ['first-aid','forklift'],
]);

$context = new FilterContext($job);

$candidates = app(Pipeline::class)
    ->send($context)
    ->through([
        StartingFilter::class,
        MatchingSkillsFilter::class,
        // Additional filters go here 
        ExceededAllowedHours::class,
    ])
    ->via('filter')
    ->then(function (FilterContext $context) {
        return $context->getCandidates();
    });
```

In this case, our first filter (`StartingFilter`) receives a `FilterContext` instance which contains no Candidates.
It performs multiple queries using SQL to build up a Collection of Candidates which are potentially eligible for the Job:

```php
class StartingFilter
{
    public function filter(FilterContext $context, Closure $next): void
    {
        // Fetch a collection of matching Candidates for this Job
        $candidates = $this->getCandidatesForJob($context->getJob());
        
        // Store them in the Context
        $context->setCandidates($candidates);
        
        // Pass the Context to the next filter in the Pipeline
        $next($context);   
    }
    
    private function getCandidatesForJob(Job $job): Collection
    {
        // Here we run a query to find any candidates which meet the basic requirements
        // for the job, such as living within a certain radius from the job location,
        // and not already being rostered onto another job at the same time.
        return Candidate::query()
            ->limit(25)
            ->get();
    } 
}
```

The next filter (`MatchingSkillsFilter`) now needs to further reduce the list of Candidates down 
to only those which have the skills required for the job:
```php
class MatchingSkillsFilter
{
    public function filter(FilterContext $context, Closure $next): void
    {
        $candidates = $this->filterCandidatesBySkills(
            $context->getCandidates,
            $context->getJob()->skills,
        );
        
        $context->setCandidates($candidates);
        
        $next($context); 
    }
    
    private function filterCandidatesBySkills(
        Collection $candidates,
        array $skills,
    ): Collection {
        if (empty($skills)) {
            // This job has no required skills
            return $candidates;
        }
        
        return $candidates
            ->filter(function(Candidate $candidate) use ($skills) {
                // Here we filter the Collection to only Candidates
                // which have ALL the $skills
            })
            ->values();
    }
}
```

For some of these filters we can apply logic conditionally, for example if the State where the job is located has different employment rules, or if the employer requires some of the checks to be skipped.

The final return value of the Pipeline is a Collection containing only Candidates that
are have passed every eligibility check.

Keeping these filters as isolated classes means we can easily add/remove/skip logic as the requirements change.
We can also implement robust test coverage by generating sample Candidates and confirming that only the eligible ones are included in the result of each filter.  

Note that in the above example our `FilterContext` holds a Collection of `Candidate` models, but in reality we use a separate DTO class (called `AvailableCandidate`) so we only need to hold the minimum data required for filtering within the Collection.
This allows us to filter thousands of Candidates in real-time without hitting against PHP memory limits.

Additionally, to ensure consistency all of our filter classes should implement the same interface, and/or extend an `AbstractFilter` class.

{% callout type="note" title="Filtering using Database Queries" %}
In our use case, the filters required querying data across third-party APIs and remote database servers so it wasn't possible to run all at once.
If all your filtering data is located within the same database schema, you could
have your filters build a single database query (each would conditionally add joins/subqueries as needed) through the pipeline.
This query can then be executed once to return your filtered results.
{% /callout %} 


## Making our Pipeline even more useful

Sometimes a specific Candidate can not be found within the results, and an Employer may want to determine why they were filtered out.
To achieve this we can add a `diagnose()` method to all of our filter classes (in addition to the `filter()` method).

For example:
```php
class OnboardingFilter implements CandidateFilterContract
{
    public function filter(FilterContext $context, Closure $next): void
    {
        $candidates = $this->filterCandidates($context);
        
        $context->setCandidates($candidates);
        
        $next($context);
    }
    
    public function diagnose(FilterContext $context, Closure $next): void
    {
        if ($this->filterCandidates($context)->isEmpty()) {
            $context->addDiagnosis(
                'The Candidate must have completed their onboarding',
            );
        }
        
        $next($context);
    }
    
    private function filterCandidates(FilterContext $context): Collection
    {
        return $context
            ->getCandidates()
            ->filter(function (Candidate $candidate) {
                return $candidate->is_onboarding_completed;
            });
    }
}
```

In the above example, our `FilterContext` contains a single `Candidate` (the one we want to diagnose).
We will run them through the `filterCandidates()` method, which will either return the Candidate or filter them out.
Our `diagnose()` method just adds a `diagnosis` to the Context containing the reason why the Candidate was filtered out.
This collection of reasons can then be displayed to the user.

```php
$context = new FilterContext($job);

// Add the Candidate to the Context
$context->setCandidates([$candidate]);

$diagnosis = app(Pipeline::class)
    ->send($context)
    ->through([
        MatchingSkillsFilter::class,
        // Additional filters go here 
        ExceededAllowedHours::class,
    ])
    ->via('diagnose')
    ->then(function (FilterContext $context) {
        return $context->getDiagnosis();
    });
```

This only requires minor changes to our filter classes so that they can run through either pipeline. 

If we wanted to take this further, we could have every filter add a diagnosis with a boolean value 
(based on whether the Candidate was filtered out), so we receive a Collection of all filters as the result:
```json
{
    "The Candidate must have completed their onboarding": true,
    "The Candidate must have a valid police check": false,
    "The Candidate must have a valid working Visa": true,
    "The Candidate must have a forklift license": true
}
```
This could easily be rendered as a checkbox list to make it clear which requirements the Candidate did not pass. 

We could add an `isApplicable()` method to our filter classes to check whether the filter applies (based on Employer configuration or the Job requirements),
so that it isn't included in the diagnosis if it's not relevant.

We could also track the number of candidates removed/remaining by each filter as it passes through
the pipeline, then use this to output statistics at the end.

The possibilities with Pipelines are endless!