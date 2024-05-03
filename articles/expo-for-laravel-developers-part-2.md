---
slug: expo-for-laravel-developers-part-2
issue: expo-for-laravel-developers-part-2
title: "Expo for Laravel Developers - Building a Mobile Todo App"
description: Pipe lines
date: 2024-05-03
category: guides
authors:
  - chris-rossi
  - mitchell-davis
published: false
---

In this post, we'll go through the minimum steps required to build a working "Todo" mobile app in Expo, 
which uses Laravel as a backend, and can be rolled out to both iOS and Android devices.

This simple mobile app will allow a Laravel User to:
1. Login using their email/password
2. Fetch their list of Tasks
3. Create a new Task
4. Mark a Task as 'Completed'
5. Logout from the app

We will show how easy it is to use Expo to build an app and have it communicate with Laravel's HTTP API, 
authenticated using Sanctum and validated using Laravel's built-in validation rules.

## Laravel API Backend

To keep things simple, we have installed a boilerplate Laravel app, and then added 
[Laravel Sanctum](https://laravel.com/docs/11.x/sanctum) to handle API authentication. 

We have then defined an additional `Task` model containing the following fields:
```php
Schema::create('tasks', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id');
    $table->string('title');
    $table->boolean('completed')->default(false);
    $table->timestamps();
});
```

We also have a `TaskController` to handle listing, creating, and updating Tasks via the API:
```php
namespace App\Http\Controllers;

use App\Http\Resources\TaskResource;
use App\Models\Task;
use Illuminate\Http\Request;

class TaskController extends Controller
{
    public function index(Request $request)
    {
        return TaskResource::collection($request->user()->tasks);
    }

    public function store(Request $request)
    {
        $data = $request->validate([
            'title' => ['required'],
        ]);

        return new TaskResource($request->user()->tasks()->create($data));
    }

    public function update(Request $request, Task $task)
    {
        if ($task->user_id !== $request->user()->id) {
            // Confirm Task is owned by User
            abort(403);
        }

        $data = $request->validate([
            'completed' => ['boolean'],
        ]);

        $task->update($data);

        return new TaskResource($task);
    }
}
```

Finally, we also have an authentication controller, routes, and JSON resources.

You can view the complete codebase for the Laravel backend here:
[https://github.com/hireatlas/laravel-todo-app](https://github.com/hireatlas/laravel-todo-app)

A new user can be created via `php artisan db:seed` for testing purposes.

We are now ready to start building our mobile app.
