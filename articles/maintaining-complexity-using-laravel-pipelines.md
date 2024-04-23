---
slug: maintaining-complexity-using-laravel-pipelines
issue: maintaining-complexity-using-laravel-pipelines
title: Maintaining complexity using Laravel pipelines
description: How we use Laravel pipelines to take control of complex processes.
date: 2024-04-23
category: guides
authors:
  - chris-rossi
image: /assets/articles/maintaining-complexity-using-laravel-pipelines.png
published: true
---

Pipelines are arguably one of Laravel's most overlooked features, but something every Laravel developer should keep in
their tool belt.

The [Laravel docs](https://laravel.com/docs/master/helpers#pipeline) only include a brief section on this feature, but
this is more than enough to get started.

Pipelines are actually used under the hood by Laravel [Middleware](https://laravel.com/docs/master/middleware), so if
you have written your own middleware you will already have some concept of the way pipelines work.

In this article, we show how we use this powerful feature to wrangle complex business processes and make our codebases
significantly easier to maintain.

## Getting started with pipelines

The Laravel `Pipeline` facade allows you to pass a value through multiple "pipes", which can then read and modify the
value as needed at each step. Each of these steps pass their output into the next step, allowing complex logic to be
broken down into simple and maintainable classes.

As an example, imagine our codebase is for a timesheet management platform, which multiple employers can use to manage
their time-and-attendance and payroll functions. We have a `Timesheet` model, and this represents an employee's shift at
work, for which they expect to get paid.

Each of these timesheets may require a review, to ensure the employee is getting paid the correct amount for their
rostered hours, with some adjustments for if they finished early or later than rostered.

To automate the payroll process as much as possible for the employers, we will implement a pipeline that runs a series
of checks on the timesheet when the employee finishes their shift.

If any of the checks fail, we need them to be recorded against the `flags` attribute. This is an array of strings with
human-readable language that will be displayed to the payroll department, so they can manually review the flagged
timesheets prior to running payroll.

We could have the following checks defined as individual classes:

1. `LongDurationsCheck`: Adds a flag if the total duration exceeds a predefined value (for example, 12 hours).
2. `MandatoryReviewCheck`: Adds a flag if the employer has enforced manual reviews of all timesheets.
3. `AutomaticallyFinishedCheck`: Adds a flag if the employee forgot to sign out from their shift, so the system
   automatically finished the shift for them.

... and so on.

### The pipeline

Our pipeline could look something like this:

```php
Pipeline::send($timesheet)
    ->through([
        LongDurationsCheck::class,
        MandatoryReviewCheck::class,
        AutomaticallyFinishedCheck::class,
    ])
    ->then(function (Timesheet $timesheet) {
        // Save the changes
        $timesheet->save();
        
        // TODO: We could dispatch a notification/email here to alert the employer
        // that there is a Timesheet requiring review
    });
```

When the employee finishes their shift, this code could be called from the controller, or more likely run in a queued
job dispatched from the controller, as some of the checks may take a bit of processing time.

Perhaps better still would be to add a `$timesheet->runChecks()` method on the `Timesheet` model directly, and that
dispatches a job to run the checks on the queue.

### Individual pipes

The pipe for checking for long durations could look like this:

```php
class LongDurationsCheck
{
    const MAX_DURATION_IN_SECONDS = 12 * 60 * 60; // 12 hours (in seconds)
    
    public function handle(Timesheet $timesheet, Closure $next): void
    {
        if ($timesheet->duration >= self::MAX_DURATION_IN_SECONDS) {
            $timesheet->addFlag('Long shift');
        }

        $next($timesheet);
    }
}
```

The pipe for mandatory reviews should only add the flag if the employer has the setting enabled:

```php
class MandatoryReviewCheck
{
    public function handle(Timesheet $timesheet, Closure $next): void
    {
        if ($timesheet->employer->requires_manual_review) {
            $timesheet->addFlag('Requires manual review');
        }

        $next($timesheet);
    }
}
```

## Using pipelines for advanced collection filtering

In one of our more complex scenarios, we need to take the details for a new job position just posted by an employer, and
then find all of the **potential** employees, which we call _candidates_, that meet all of the criteria.

This requires more than 10 different checks to be performed in order to determine if each candidate is eligible for that
specific job. To make this process more manageable, we create and use a custom `FilterContext` class to hold all the
required data for the filtering.

```php
class FilterContext
{
    private Collection $candidates;
    
    public function __construct(
        private Job $job,
    ) {
        $this->candidates = collect();
    }
    
    public function getJob(): Job
    {
        return $this->job;
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
}
```

Any time we need to get the list of available candidates for a given job, we create an instance of the `FilterContext`
and pass it through our pipeline:

```php
// Create a job with a start/finish time, a location, and several mandatory skills.
// You can imagine that this information came in from a HTTP request, and the Job was created inside a controller.
$job = Job::create([
    'title' => 'Lead Forklift Driver',
    'start' => '2024-05-01 09:00:00',
    'finish' => '2024-05-01 17:00:00',
    'location_id' => 12345,
    'skills' => [
         // First aid training and forklift driving are mandatory for this job.
         'first-aid',
         'forklift',
    ],
]);

$context = new FilterContext($job);

$candidates = Pipeline::send($context)
    ->through([
        // At this point, there are no candidates in our context, so the StartingFilter will populate it with every 
        // possible available candidate.
        StartingFilter::class,
        
        // At this point, the context now has every possible available candidate, so each of the following filters just 
        // remove candidates based on more refined eligibility logic.
        MatchingSkillsFilter::class,
        ProximityToLocationFilter::class,
        ExceededAllowedHoursFilter::class,
        
        // Additional filters go here...
    ])
    ->via('filter')
    ->then(function (FilterContext $context) {
        return $context->getCandidates();
    });
```

In this case, our first filter `StartingFilter` receives a `FilterContext` instance which contains no candidates.

It performs multiple queries using SQL to build up a collection of candidates which are potentially eligible for the
job:

```php
class StartingFilter
{
    public function filter(FilterContext $context, Closure $next): void
    {
        // Fetch a collection of matching candidates for this job.
        $candidates = $this->getCandidatesForJob($context->getJob());
        
        // Store them in the context.
        $context->setCandidates($candidates);
        
        // Pass the context to the next filter in the pipeline.
        $next($context);   
    }
    
    private function getCandidatesForJob(Job $job): Collection
    {
        // Here we run a query to find any candidates which meet the basic requirements
        // for the job, such as being active within the system, and not already being rostered onto another job at the 
        // same time.
        
        // For illustrative purposes, we are just fetching 25 candidates from the database, but you would write your 
        // own logic here.
        return Candidate::query()
            ->limit(25)
            ->get();
    } 
}
```

The next filter, `MatchingSkillsFilter`, now needs to further reduce the list of Candidates down
to only those which have the skills required for the job:

```php
class MatchingSkillsFilter
{
    public function filter(FilterContext $context, Closure $next): void
    {
        // Run the filtering logic defined in this class to reduce our collection of candidates to just those that have 
        // matching skills.
        $candidates = $this->filterCandidatesBySkills(
            $context->getCandidates(),
            $context->getJob()->skills,
        );
        
        // Store them in the context.
        $context->setCandidates($candidates);
        
        // Pass the context to the next filter in the pipeline.
        $next($context); 
    }
    
    private function filterCandidatesBySkills(Collection $candidates, array $skills): Collection 
    {
        if (empty($skills)) {
            // This job has no required skills.
            return $candidates;
        }
        
        return $candidates
            ->filter(function(Candidate $candidate) use ($skills) {
                // Here we filter the collection to only candidates which have ALL of the $skills.
            })
            ->values();
    }
}
```

For some of these filters we can apply logic conditionally. An example is where, if the state where the job is located,
has different employment rules from the rest of the country, we can conditionally run or not run the logic related to
those employment rules.

The final return value of the pipeline is a collection containing only candidates that are have passed every eligibility
check.

```php
$context = new FilterContext($job);

$candidates = Pipeline::send($context) // [tl! focus]
    ->through([
        // At this point, there are no candidates in our context, so the StartingFilter will populate it with every 
        // possible available candidate.
        StartingFilter::class,
        
        // At this point, the context now has every possible available candidate, so each of the following filters just 
        // remove candidates based on more refined eligibility logic.
        MatchingSkillsFilter::class,
        ProximityToLocationFilter::class,
        ExceededAllowedHoursFilter::class,
        
        // Additional filters go here...
    ])
    ->via('filter')
    ->then(function (FilterContext $context) { // [tl! focus]
        return $context->getCandidates(); // [tl! focus]
    }); // [tl! focus]
```

Keeping these filters as isolated classes means we can easily add/remove/skip logic as the requirements change.

We can also implement robust test coverage by generating sample candidates, and confirming that only the eligible ones
are included in the result of each filter.

### Improvements and considerations

- **Consider reducing the size of your data**

  Note that in the above example our `FilterContext` holds a collection of `Candidate` models, but in reality we use a
  separate DTO class called `AvailableCandidate`. We do this to ensure that we only keep in memory the minimum data
  required for filtering within the collection, such as the id, name, address, and status.

  This allows us to filter thousands of candidates in real-time without hitting against PHP memory limits.

- **Add an interface and optionally an abstract class**

  This will ensure consistency and enforce that all of the filter classes implement the same interface, and/or extend
  an `AbstractFilter` class.

- **Do as much logic in the database as possible**

  In our use case, the filters need to query data across third-party APIs and remote database servers, so it wasn't
  possible to run all of the filtering logic in one giant query.

  If all your filtering data is located within the same database schema, you could have your filters build a single
  database query (each would conditionally add joins/subqueries as needed) through the pipeline. This query can then be
  executed at the end of your pipeline to return your filtered results with a single query.

  For more on this, take a look at [Eloquent Performance Patterns](https://eloquent-course.reinink.ca/) by Jonathan
  Reinink.

## Making our pipeline even more useful

Sometimes a specific candidate can not be found within the filtered results, and an employer may want to determine why
they were filtered out.

To achieve this we can add a `diagnose()` method to all of our filter classes, in addition to the `filter()` method.

```php
class OnboardingFilter
{
    public function filter(FilterContext $context, Closure $next): void
    {
        $candidates = $this->filterCandidates($context);
        
        $context->setCandidates($candidates);
        
        $next($context);
    }
    
    public function diagnose(FilterContext $context, Closure $next): void // [tl! focus]
    { // [tl! focus]
        if ($this->filterCandidates($context)->isEmpty()) { // [tl! focus]
            $context->addDiagnosis('The candidate must have completed their onboarding.'); // [tl! focus]
        } // [tl! focus]
         // [tl! focus]
        $next($context); // [tl! focus]
    } // [tl! focus]
    
    private function filterCandidates(FilterContext $context): Collection
    {
        return $context
            ->getCandidates()
            ->filter(function (Candidate $candidate) {
                return $candidate->is_onboarding_completed;
            })
            ->values();
    }
}
```

In the above example, our `FilterContext` contains a single `Candidate`, which is the one we want to diagnose.

We will run them through the `filterCandidates()` method, which will either return the candidate or filter them out.

Our `diagnose()` method just adds a `diagnosis` to the context containing the reason why the candidate was filtered out.

This collection of reasons can then be displayed to the user.

```php
$context = new FilterContext($job);

// Add the candidate to the context. [tl! focus]
$context->setCandidates(collect([$candidate])); // [tl! focus]

$diagnosis = Pipeline::send($context) // [tl! focus]
    ->through([
        StartingFilter::class,
        MatchingSkillsFilter::class,
        ProximityToLocationFilter::class,
        ExceededAllowedHoursFilter::class,
        
        // Additional filters go here...
    ])
    ->via('diagnose') // Note that this time we call the diagnose method, not the filter method. [tl! focus]
    ->then(function (FilterContext $context) { // [tl! focus]
        return $context->getDiagnosis(); // [tl! focus]
    }); // [tl! focus]
```

This only requires minor changes to our filter classes so that they can run through either pipeline.

### Taking it further

- **Rendering a checklist for the employer**

  We could have every filter add a diagnosis with a boolean value, based on whether the candidate was filtered out, so
  we receive a collection of all filters as the result:

  ```json
  {
    "The candidate must have completed their onboarding.": true,
    "The candidate must have a valid police check.": false,
    "The candidate must have a valid working visa.": true,
    "The candidate must have a forklift license.": true
  }
  ```

  This could easily be rendered as a checkbox list to make it clear to the employer which requirements the candidate did
  not pass.

- **Conditionally applying filters**

  We could add an `isApplicable()` method to our filter classes to check whether the filter applies, based on employer
  settings or the job requirements, so that it isn't included in the diagnosis if it's not relevant.

- **Tracking totals for each filter**

  For analytical purposes, we could also track the number of candidates still remaining at the end of each filter as it
  passes through the pipeline, and then use this to output statistics at the end.

The possibilities with pipelines are endless!