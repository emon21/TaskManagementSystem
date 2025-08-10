# Task মানাগমেন্ট সিস্টেম

-   দারুন — চলি শুরু করি। নিচে একটা Task Management System (Laravel 10 + Blade) এর সম্পূর্ণ উচ্চ-স্তরের ফিচার লিস্ট, ডাটাবেস ডিজাইন (টেবিল/কলাম/রিলেশন) এবং প্রয়োগ করার জন্য প্রয়োজনীয় প্রোগ্রামিং লজিক, রুট/কন্ট্রোলার/মডেল/ব্লেড সিনট্যাক্সের ছোট ছোট উদাহরণ দেওয়া আছে। আপনি চাইলে আমি এগুলো থেকে মাইগ্রেশন, কন্ট্রোলার, ব্লেড ফাইল সম্পূর্ণ করে দিতে পারি — বললেই শুরু করি।

ফিচার লিস্ট (মডিউল ভিত্তিতে অন্যতম গুরুত্বপূর্ণ আইটেম)
ইউজার অথেনটিকেশন (Laravel Breeze/Jetstream) — Register/Login, Roles & Permissions (Admin, Manager, Member)

প্রজেক্ট/টিম ম্যানেজমেন্ট — প্রজেক্ট বানানো, টিম সদস্য যোগ করা

টাস্ক CRUD — create/read/update/delete

অ্যাসাইনমেন্ট — টাস্ককে একাধিক ইউজার বা এক ইউজারকে অ্যাসাইন করা (assignee)

স্ট্যাটাস ও প্রারিটি — To Do / In Progress / Review / Done, Low/Medium/High

ডেডলাইন ও রিমাইন্ডার — due_date, email/notification reminders

সাব-টাস্ক ও চেকলিস্ট

কমেন্ট ও অ্যাক্টিভিটি লগ

ফাইল অ্যাটাচমেন্টস (uploads)

ট্যাগ/ক্যাটাগরি/ফিল্টারিং/সার্চ

ফিল্টারেড ভিউ ও কাস্টম বোর্ড (Kanban board)

নোটিফিকেশন (database & mail)

টাস্ক টাইম ট্র্যাকিং (optional)

রিপোর্টস ও এক্সপোর্ট (CSV/PDF)

রোল-বেইজড অথরাইজেশন (Policy/Gate)

API endpoints (mobile/SPA support)

ডাটাবেস ডিজাইন (টেবিল ও কলাম) — ER ধারণা পরে আছে
নীচে মূল টেবিলগুলো — প্রাথমিক ও প্রয়োজনীয় কলামসহ (timestamps implied যেখানে দরকার):

users
id (bigIncrements)

name (string)

email (string, unique)

password (string)

role (enum: admin, manager, member) — বা আলাদা roles টেবিল

avatar (string, nullable)

timestamps

projects
id

name

description (text, nullable)

owner_id (foreign users.id)

status (enum: active, archived)

start_date (date, nullable)

end_date (date, nullable)

timestamps

project_user (pivot) — project members
id

project_id (fk)

user_id (fk)

role (string) — e.g. 'member','pm'

joined_at (timestamp)

timestamps

tasks
id

project_id (nullable) — fk

parent_id (nullable) — for sub-task (self fk)

title (string)

description (text, nullable)

creator_id (fk users.id)

priority (enum: low, medium, high, urgent)

status (enum: todo, in_progress, review, done, blocked)

start_at (datetime, nullable)

due_date (datetime, nullable)

estimated_time (integer minutes nullable)

time_spent (integer minutes default 0)

is_recurring (boolean) // optional

recurrence_rule (string/json nullable)

attachments_count (integer default 0)

comment_count (integer default 0)

timestamps

softDeletes() optional

task_user (assignments) — a task can have many assignees
id

task_id

user_id

assigned_by (user id)

assigned_at

role (optional string)

timestamps

comments
id

task_id (fk)

user_id (fk)

body (text)

parent_id nullable (reply)

edited_at nullable

timestamps

attachments
id

task_id (fk)

user_id (uploader)

filename

path

mimetype

size

timestamps

tags
id

name

color (string nullable)

timestamps

tag_task (pivot)
tag_id

task_id

activities (activity log)
id

user_id (who did it, nullable)

subject_type (morph) — e.g., App\Models\Task

subject_id

description (string) — e.g., "changed status from X to Y"

metadata (json nullable)

created_at

notifications (use laravel notifications)
use built-in notifications table if needed

timers (for time tracking) optional
id

task_id

user_id

started_at

stopped_at

duration (minutes)

timestamps

রিলেশনস: users ↔ projects (many-to-many via project_user), projects ↔ tasks (one-to-many), tasks ↔ users (many-to-many via task_user), tasks ↔ comments (one-to-many), tasks ↔ attachments (one-to-many), tasks ↔ tags (many-to-many).

Laravel 10 — মাইগ্রেশন, মডেল ও কন্ট্রোলার লজিক (সংক্ষিপ্ত উদাহরণ)
Example migration (tasks)
php
Copy
Edit
// database/migrations/xxxx_create_tasks_table.php
Schema::create('tasks', function (Blueprint $table) {
$table->id();
$table->foreignId('project_id')->nullable()->constrained()->onDelete('set null');
$table->foreignId('parent_id')->nullable()->constrained('tasks')->onDelete('cascade');
$table->string('title');
$table->text('description')->nullable();
$table->foreignId('creator_id')->constrained('users')->onDelete('cascade');
$table->enum('priority', ['low','medium','high','urgent'])->default('medium');
$table->enum('status', ['todo','in_progress','review','done','blocked'])->default('todo');
$table->timestamp('start_at')->nullable();
$table->timestamp('due_date')->nullable();
$table->integer('time_spent')->default(0);
$table->timestamps();
$table->softDeletes();
});
Task Model (app/Models/Task.php)
php
Copy
Edit
class Task extends Model
{
use HasFactory, SoftDeletes;

    protected $fillable = [
        'project_id','parent_id','title','description','creator_id',
        'priority','status','start_at','due_date','time_spent'
    ];

    public function project(){ return $this->belongsTo(Project::class); }
    public function creator(){ return $this->belongsTo(User::class,'creator_id'); }
    public function assignees(){ return $this->belongsToMany(User::class,'task_user')->withTimestamps(); }
    public function comments(){ return $this->hasMany(Comment::class); }
    public function attachments(){ return $this->hasMany(Attachment::class); }
    public function tags(){ return $this->belongsToMany(Tag::class,'tag_task'); }
    public function parent(){ return $this->belongsTo(Task::class,'parent_id'); }
    public function subtasks(){ return $this->hasMany(Task::class,'parent_id'); }

}
TaskController (key methods)
php
Copy
Edit
class TaskController extends Controller
{
public function index(Request $req){
        // filter by project, status, assignee, tag
        $query = Task::with(['assignees','project'])
            ->when($req->project_id, fn($q)=>$q->where('project_id',$req->project_id))
            ->when($req->status, fn($q)=>$q->where('status',$req->status));
$tasks = $query->paginate(20);
return view('tasks.index', compact('tasks'));
}

    public function store(Request $req){
        $data = $req->validate([
            'title'=>'required|string|max:255',
            'description'=>'nullable|string',
            'project_id'=>'nullable|exists:projects,id',
            'due_date'=>'nullable|date',
            'assignees'=>'nullable|array'
        ]);
        $data['creator_id'] = auth()->id();
        $task = Task::create($data);
        if(!empty($req->assignees)){
            $task->assignees()->sync($req->assignees); // assign users
        }
        // activity log & notifications
        activity()->log("Task created: {$task->title}");
        Notification::send($task->assignees, new TaskAssigned($task));
        return redirect()->route('tasks.show',$task)->with('success','Task created');
    }

    public function update(Request $req, Task $task){
        $data = $req->validate([ 'title'=>'required','status'=>'required' ]);
        $old = $task->getOriginal();
        $task->update($data);
        // if status changed -> log & notify
        if($old['status'] !== $task->status){
            activity()->causedBy(auth()->user())->performedOn($task)
                ->withProperties(['from'=>$old['status'],'to'=>$task->status])
                ->log('status_changed');
            $task->assignees->each->notify(new TaskStatusChanged($task, $old['status']));
        }
        return back()->with('success','Updated');
    }

    public function destroy(Task $task){
        $task->delete();
        activity()->log("Task deleted: {$task->title}");
        return back()->with('success','Deleted');
    }

}
Blade সিনট্যাক্স — কিছুকিছু উদাহরণ

1. টাস্ক তালিকা (resources/views/tasks/index.blade.php)
   blade
   Copy
   Edit
   @extends('layouts.app')

@section('content')

<h1>Tasks</h1>

<form method="GET" class="mb-3">
  <select name="status" onchange="this.form.submit()">
    <option value="">All</option>
    <option value="todo" {{ request('status')=='todo' ? 'selected' : '' }}>To Do</option>
    <option value="in_progress" {{ request('status')=='in_progress' ? 'selected' : '' }}>In Progress</option>
    <option value="done" {{ request('status')=='done' ? 'selected' : '' }}>Done</option>
  </select>
</form>

<div class="task-list">
  @foreach($tasks as $task)
    <div class="card mb-2">
      <div class="card-body d-flex justify-content-between">
        <div>
          <h5><a href="{{ route('tasks.show',$task) }}">{{ $task->title }}</a></h5>
          <p class="mb-0 text-muted">Project: {{ $task->project?->name ?? '—' }} · Due: {{ $task->due_date?->format('d M Y') ?? 'N/A' }}</p>
          <p class="mb-0">
            Assignees:
            @foreach($task->assignees as $u)
              <span class="badge bg-secondary">{{ $u->name }}</span>
            @endforeach
          </p>
        </div>
        <div>
          <span class="badge @if($task->status=='done') bg-success @else bg-info @endif">{{ ucfirst($task->status) }}</span>
        </div>
      </div>
    </div>
  @endforeach
</div>

{{ $tasks->links() }}
@endsection 2) টাস্ক ক্রিয়েট ফর্ম (resources/views/tasks/create.blade.php)
blade
Copy
Edit
@extends('layouts.app')
@section('content')

<form action="{{ route('tasks.store') }}" method="POST">
  @csrf
  <div class="mb-3">
    <label>Title</label>
    <input type="text" name="title" value="{{ old('title') }}" class="form-control" required>
  </div>
  <div class="mb-3">
    <label>Description</label>
    <textarea name="description" class="form-control">{{ old('description') }}</textarea>
  </div>
  <div class="mb-3">
    <label>Assignees</label>
    <select name="assignees[]" multiple class="form-control">
      @foreach(\App\Models\User::all() as $user)
        <option value="{{ $user->id }}">{{ $user->name }}</option>
      @endforeach
    </select>
  </div>
  <button class="btn btn-primary">Create</button>
</form>
@endsection
3) টাস্ক শো পেজে কমেন্ট ও অ্যাকশন (resources/views/tasks/show.blade.php)
blade
Copy
Edit
@extends('layouts.app')
@section('content')
<h2>{{ $task->title }}</h2>
<p>{{ $task->description }}</p>

@include('tasks.partials.\_comments', ['comments' => $task->comments])

<form action="{{ route('tasks.comment',$task) }}" method="POST">
  @csrf
  <textarea name="body" class="form-control" required></textarea>
  <button class="btn btn-sm btn-primary mt-2">Comment</button>
</form>
@endsection
অথরাইজেশন ও পলিসি (সংক্ষিপ্ত)
TaskPolicy তৈরি করে view, update, delete, assign মেথড সংজ্ঞায়িত করবেন। কন্ট্রোলারে authorize() বা Gate::allows() ব্যবহার করুন।

উদাহরণ: only creator, project owner, or assigned user can update status (business rule) — enforce in policy.

নোটিফিকেশন ও অ্যাক্টিভিটি
Laravel Notifications (Mail, Database, Broadcast) ব্যবহার করুন — php artisan make:notification TaskAssigned

Activity logging: spatie/laravel-activitylog প্যাকেজ কাজে লাগালে অনেক সহজ হয়।

রুট উদাহরণ (routes/web.php)
php
Copy
Edit
Route::middleware(['auth'])->group(function(){
Route::resource('tasks', TaskController::class);
Route::post('tasks/{task}/comment', [TaskController::class,'comment'])->name('tasks.comment');
Route::post('tasks/{task}/assign', [TaskController::class,'assign'])->name('tasks.assign');
});
উন্নত লজিক আইডিয়া (business rules)
যদি টাস্ক স্ট্যাটাস done হয়, সেটি auto-archive না করলে time_spent, completed_at সেট করা হবে।

সাব-টাস্কসমূহ সব completed হলে প্যারেন্ট টাস্কের স্ট্যাটাস auto done করা যেতে পারে (configurable).

ডিউ ডেটে যদি ওভারডিউ হয় → cron job / schedule:daily চালিয়ে overdue_flag set করে নোটিফিকেশন পাঠানো।

Kanban বোর্ড: status কলামগুলোর জন্য Ajax-based drag & drop — update API endpoint যা ইউনিট-অফার্ড করে অবস্থান্তর করে।

Permission matrix: Admin = সবকিছু, Manager = প্রকল্পে CRUD, Member = assignকৃত টাস্ক দেখবো এবং update করবে।

ডেভেলপমেন্ট টিপস
Use Form Requests (php artisan make:request TaskStoreRequest) — validation logic clean রাখবে।

Eager load relations on index to avoid N+1 (with(['assignees','project'])).

Use policies & gates early — security holes কমে যায়।

Store files using Laravel filesystem (S3/local) and keep only paths in DB.

Tests: Feature tests for task flows (create, assign, status change)।

যদি চান আমি এখন থেকে যে কোনো এক বা একাধিক জিনিস পুরো করে দিলাম — যেমন:

সম্পূর্ণ মাইগ্রেশন ফাইল + Model + Factory + Seeder (প্রকল্পের জন্য), অথবা

TaskController সম্পূর্ণ CRUD + রিসোর্স ভিউ (Blade) + রুট, অথবা

Kanban বোর্ডের জন্য Ajax + Vue/Alpine + Blade ইন্টারগ্রেশন — আপনার পছন্দে কাজ শুরু করি।

বলুন, কোন অংশটা এখনই চান ডিটেইলে (মাইগ্রেশন/কন্ট্রোলার/ব্লেড/পলিসি/নোটিফিকেশন)?

## About Laravel

Laravel is a web application framework with expressive, elegant syntax. We believe development must be an enjoyable and creative experience to be truly fulfilling. Laravel takes the pain out of development by easing common tasks used in many web projects, such as:

-   [Simple, fast routing engine](https://laravel.com/docs/routing).
-   [Powerful dependency injection container](https://laravel.com/docs/container).
-   Multiple back-ends for [session](https://laravel.com/docs/session) and [cache](https://laravel.com/docs/cache) storage.
-   Expressive, intuitive [database ORM](https://laravel.com/docs/eloquent).
-   Database agnostic [schema migrations](https://laravel.com/docs/migrations).
-   [Robust background job processing](https://laravel.com/docs/queues).
-   [Real-time event broadcasting](https://laravel.com/docs/broadcasting).

Laravel is accessible, powerful, and provides tools required for large, robust applications.

## Learning Laravel

Laravel has the most extensive and thorough [documentation](https://laravel.com/docs) and video tutorial library of all modern web application frameworks, making it a breeze to get started with the framework.

You may also try the [Laravel Bootcamp](https://bootcamp.laravel.com), where you will be guided through building a modern Laravel application from scratch.

If you don't feel like reading, [Laracasts](https://laracasts.com) can help. Laracasts contains thousands of video tutorials on a range of topics including Laravel, modern PHP, unit testing, and JavaScript. Boost your skills by digging into our comprehensive video library.

## Laravel Sponsors

We would like to extend our thanks to the following sponsors for funding Laravel development. If you are interested in becoming a sponsor, please visit the [Laravel Partners program](https://partners.laravel.com).

### Premium Partners

-   **[Vehikl](https://vehikl.com/)**
-   **[Tighten Co.](https://tighten.co)**
-   **[WebReinvent](https://webreinvent.com/)**
-   **[Kirschbaum Development Group](https://kirschbaumdevelopment.com)**
-   **[64 Robots](https://64robots.com)**
-   **[Curotec](https://www.curotec.com/services/technologies/laravel/)**
-   **[Cyber-Duck](https://cyber-duck.co.uk)**
-   **[DevSquad](https://devsquad.com/hire-laravel-developers)**
-   **[Jump24](https://jump24.co.uk)**
-   **[Redberry](https://redberry.international/laravel/)**
-   **[Active Logic](https://activelogic.com)**
-   **[byte5](https://byte5.de)**
-   **[OP.GG](https://op.gg)**

## Contributing

Thank you for considering contributing to the Laravel framework! The contribution guide can be found in the [Laravel documentation](https://laravel.com/docs/contributions).

## Code of Conduct

In order to ensure that the Laravel community is welcoming to all, please review and abide by the [Code of Conduct](https://laravel.com/docs/contributions#code-of-conduct).

## Security Vulnerabilities

If you discover a security vulnerability within Laravel, please send an e-mail to Taylor Otwell via [taylor@laravel.com](mailto:taylor@laravel.com). All security vulnerabilities will be promptly addressed.

## License

The Laravel framework is open-sourced software licensed under the [MIT license](https://opensource.org/licenses/MIT).
