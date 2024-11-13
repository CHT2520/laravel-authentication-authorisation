# Authentication and Authorisation

This practical provides an overview of implementing authentication and authorisation using Laravel. As always, you should refer to the Laravel documentation (https://laravel.com/) for complete information and Laracasts (https://laracasts.com/series/30-days-to-learn-laravel-11) for additional advice.

The starting point for this practical is the **Intro to Laravel** practical we did a couple of weeks ago (you'd also be fine using the Eloquent Relationships example as a starting point).

-   If you've completed this work you'll probably need to change the _DocumentRoot_ settings in your _httpd.conf_ file to set _film-app_ example as the web server root e.g.

    ```
    DocumentRoot "/xampp/htdocs/film-app/public"
    <Directory "/xampp/htdocs/film-app/public">
    ```

-   If you don't have your own copy of this work you can get a copy from https://github.com/CHT2520/intro-to-laravel-code.

## Authentication

### The Database

When we create a new Laravel project a _users_ table migration is created for us. Look under _databases/migrations_ and you should see this. The first thing we need to do is populate this table with some example users.

#### Seeding the Users Table

-   Using Artisan create a new seeder for the _users_ table.

```
php artisan make:seeder UserSeeder
```

-   Add some example users e.g.

```php
namespace Database\Seeders;

use Illuminate\Support\Facades\DB;
use Illuminate\Database\Console\Seeds\WithoutModelEvents;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash;

class UserSeeder extends Seeder
{
    /**
     * Run the database seeds.
     */
    public function run(): void
    {
        DB::table('users')->insert(['name' => 'Kate Hutton', 'email' => 'k.l.hutton@hudstudent.ac.uk', 'password' => Hash::make('password')]);
        DB::table('users')->insert(['name' => 'Yousef Miandad', 'email' => 'y.miandad@hudstudent.ac.uk', 'password' => Hash::make('letmein')]);
        DB::table('users')->insert(['name' => 'Sunil Laxman', 'email' => 's.laxman@hudstudent.ac.uk', 'password' => Hash::make('password2')]);
    }
}
```

-   Most of this should be self-explanatory. The only new thing here is the use of the `Hash` class to generate a hash for each user's password.
-   Add the _UserSeeder_ to the list of seeders in _DatabaseSeeder.php_.

```php
    public function run(): void
    {
        $this->call([
            FilmSeeder::class,
            UserSeeder::class
        ]);
    }
```

-   Ask Artisan to re-generate the database tables.

```
- php artisan migrate:fresh --seed
```

-   In phpMyAdmin check you have some rows in the _users_ table and notice that the passwords have been hashed.

### Implementing Authentication

#### Creating a Log In View

-   Create a new folder in the _resources/views_ folder, name it 'auth'.
-   Create a new file in the _auth_ folder called _login.blade.php_.
-   Add the following code into this file

```html
<x-layout title="Sign In">
    <h1>Sign In</h1>
    <form method="POST" action="/login">
        @csrf
        <div>
            <label for="email">Email:</label>
            <input type="text" id="email" name="email" />
        </div>
        <div>
            <label for="password">Password:</label>
            <input type="password" id="password" name="password" />
        </div>
        <div>
            <button type="submit">Sign in</button>
        </div>
    </form>
</x-layout>
```

#### Creating a Controller and Route

-   Using Artisan create a new controller to handle authentication

```
php artisan make:controller AuthController
```

-   Add an `index()` method to the controller

```php
function index()
    {
        return view('auth.login');
    }
```

-   In the _routes/web.php_ file add a route for the login form i.e.

```php
Route::get('/login', [AuthController::class, 'index']);
```

-   You will also need to make sure you have imported the `AuthController` (look at the top of the file to see how we did this for the `FilmController`)

-   Test this works by entering http://localhost/login into a browser. You should see the login form.

#### Testing the Log In Details

-   Look in _login.blade.php_, take note of the _action_ and _method_.
-   In _web.php_ add a route to handle the log in form submission.

```php
Route::post('/login', [AuthController::class, 'login']);
```

-   In `AuthController` import the `Auth` facade at the top of the file.

```php
use Illuminate\Support\Facades\Auth;
```

-   Then add a `login()` method to handle the _login_ route

```php
function login(Request $request)
{
    $userDetails = [
        "email" => $request->email,
        "password" => $request->password
    ];

    if (Auth::attempt($userDetails)) {
        $request->session()->regenerate();
        return redirect('/films');
    }
    return back();
}
```

-   The first part of this method simply gets the user details from the form and puts them into an array.
-   The `Auth` facade performs authentication for us. The `attempt()` method will check these details against the _users_ table in the database. If we get a match we want to start a new session `$request->session()->regenerate();` and redirect the user to the homepage of the app.
-   If the login attempt is unseccessfull we 'fall through' to the final line which sends the user back to the log in form.
-   Test this works, look in the _UserSeeder.php_ file for usernames and passwords to try.

#### Giving the User Some Feedback

-   Even though this works it isn't clear to the user that they have successfully signed in
-   Open _resources/views/components/layout.blade.php_
-   Add the following code immediately after the opening `<body>` tag and before the `<nav>`

```php
@auth
    <div>
        Logged in as {{Auth::user()->name}}
        <form method='POST' action='/logout'>
            @csrf
            <button type='submit'>Logout</button>
        </form>
    </div>
@endauth
```

The `@auth` blade directive is used to markout a block of code that is only for authenticated (logged in) users.

-   Login into the website and you should see that the username is displayed at the top of the page.

#### Implementing Logout

-   The form in this block of code submits to a URI of `/logout`.

-   Add a route for this URI

```php
Route::post('/logout', [AuthController::class, 'logout']);
```

-   Add a `logout()` method to the `AuthController`.

```php
function logout(Request $request)
{
    Auth::logout();
    $request->session()->invalidate();
    $request->session()->regenerateToken();
    return redirect('/login');
}
```

-   This simply logs the user out using the `Auth` facade, cleans up the session data and redirects them to the log in page.
-   Test this works.

#### Providing a Sign In Link

-   Back in _layout.blade.php_ modify the navigation options to include a sign-in option that will only be displayed for users that haven't logged in.
    -   The `@guest` directive marks out code that will only be displayed for users that aren't authenticated.

```html
<nav>
    <ul>
        <li><a href="/films">Home</a></li>
        <li><a href="/films/create">Add new film</a></li>
        <li><a href="/films/about">About</a></li>
        @guest
        <li><a href="/login">Sign in</a></li>
        @endguest
    </ul>
</nav>
```

-   Again test this works

### Protecting Routes

We want to prevent guest users from accessing certain pages in our site.

-   First, in _index.blade.php_ we'll display different content depending on whether or not the user is logged in.
    -   Edit this file so it looks like the following

```html
<x-layout title="List the films">
    <h1>Here's a list of films</h1>
    @auth @foreach ($films as $film)
    <p>
        <a href="/films/{{$film->id}}"> {{$film->title}} </a>
    </p>
    @endforeach @endauth @guest
    <p>
        You need to be logged in to view the content of this website. @endguest
    </p></x-layout
>
```

-   This should make sense, it simply uses the `@auth` and `@guest` directives to display different content for different users.
-   Check this works.

Next we will stop guest users from accessing certain routes. Open _routes/web.php_ and add route middleware to prevent guest users from accessing the create, show and edit pages. You may have additional routes e.g. for update and delete. You should protect these as well.

```php
use App\Http\Controllers\FilmController;
use App\Http\Controllers\AuthController;
use Illuminate\Support\Facades\Route;

Route::get('/films', [FilmController::class, 'index']);
Route::get('/films/about', [FilmController::class, 'about']);
Route::get('/films/create', [FilmController::class, 'create'])->middleware('auth');
Route::post('/films', [FilmController::class, 'store'])->middleware('auth');
Route::get('/films/{id}', [FilmController::class, 'show'])->middleware('auth');
Route::get('/films/{id}/edit', [FilmController::class, 'edit'])->middleware('auth');


Route::get('/login', [AuthController::class, 'index'])->name("login");
Route::post('/login', [AuthController::class, 'login']);
Route::post('/logout', [AuthController::class, 'logout']);
```

-   Make sure you are logged out and enter the following URI http://localhost/films/create, you should find you get redirected to the login page.

## Authorisation Using Gates

Often we want to provide different levels of access for different types of users. To keep things simple we'll have two types of user, a regular user and an admin user.

### Setting up the database

We'll store a _role_id_ for each user in the _users_ database table.

-   Open the migration for the _users_ table from the _database/migrations_ folder.
-   Edit this file to add a _role_id_ column. The function for creating the _users_ table will then look like:-

```php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->tinyInteger('role_id'); //this is the new line
    $table->rememberToken();
    $table->timestamps();
});
```

-   Modify the _UserSeeder.php_ file to include a _role_id_ for each user

```php
public function run(): void
{
    DB::table('users')->insert(['name' => 'Kate Hutton', 'email' => 'k.l.hutton@hudstudent.ac.uk', 'password' => Hash::make('password'), 'role_id' => 2]);
    DB::table('users')->insert(['name' => 'Yousef Miandad', 'email' => 'y.miandad@hudstudent.ac.uk', 'password' => Hash::make('letmein'), 'role_id' => 2]);
    DB::table('users')->insert(['name' => 'Sunil Laxman', 'email' => 's.laxman@hudstudent.ac.uk', 'password' => Hash::make('password2'), 'role_id' => 1]);
}
```

-   Re-run the migrations and seeding

```
php artisan migrate:fresh --seed
```

-   Check phpMyAdmin to make sure this has worked.

### Adding a gate

-   Add a gate to the `boot()` method of _app/Providers/ServiceProvider.php_.
    -   A gate is simply a function that decides if a user is allowed to perform a certain action. This gate simply checks if the user as a _role_id_ of 2.

```php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\Gate;
use App\Models\User;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        //
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Gate::define('edit', function (User $user) {
            return $user->role_id === 2;
        });
    }
}
```

### Using the Gate

There are a number of different ways of using the gate - in a controller, a route, a view. We will use it in our routes file.

-   Modify the _web.php_ so that only users that pass the _edit_ gate can access the create, store and edit routes.

```php
use App\Http\Controllers\FilmController;
use App\Http\Controllers\AuthController;
use Illuminate\Support\Facades\Route;

Route::get('/films', [FilmController::class, 'index']);
Route::get('/films/about', [FilmController::class, 'about']);
Route::get('/films/create', [FilmController::class, 'create'])->middleware(['auth', 'can:edit']);
Route::post('/films', [FilmController::class, 'store'])->middleware(['auth', 'can:edit']);
Route::get('/films/{id}', [FilmController::class, 'show'])->middleware('auth');
Route::get('/films/{id}/edit', [FilmController::class, 'edit'])->middleware(['auth', 'can:edit']);


Route::get('/login', [AuthController::class, 'index'])->name("login");
Route::post('/login', [AuthController::class, 'login']);
Route::post('/logout', [AuthController::class, 'logout']);
```

-   Test this works
-   Log into the app as s.laxman@hudstudent.ac.uk (role_id of 1) and try to add a new film. You should be denied access.

### Using the @can Directive in Views

This works but really we want hide navigation options from users that don't have the authorisation to perform those actions.

-   Open _layout.blade_.
-   Modify the navigation bar to use the `@can` blade directive

```html
<nav>
    <ul>
        <li><a href="/films">Home</a></li>
        @can('edit')
        <li><a href="/films/create">Add new film</a></li>
        @endcan
        <li><a href="/films/about">About</a></li>
        @guest
        <li><a href="/login">Sign in</a></li>
        @endguest
    </ul>
</nav>
```

-   Open _show.blade.php_ and hide the edit and delete buttons for non-authorised users.

```html
<x-layout title="Show the details for a film">
    <h1>{{$film->title}}</h1>
    <p>Year:{{$film->year}}</p>
    <p>Duration:{{$film->duration}}</p>

    @can('edit')
    <a href="/films/{{$film->id}}/edit">
        <button>Edit</button>
    </a>

    <form method="POST" action="/films">
        @csrf @method('DELETE')
        <input type="hidden" name="id" value="{{$film->id}}" />
        <button type="submit">Delete</button>
    </form>
    @endcan
</x-layout>
```

-   Test this works and take a moment to understand the different levels of access for the app.

### Adding a custom 403 page

When a user isn't authorised to perform an action they receive a 403 page. It would be nice if this page was styled to fit with the rest of the application

-   Create a new folder called 'errors' in the _resources/views_ folder.
-   Create a new _404.blade.php_ file in this folder.
-   Add the following code in this page.

```html
<!DOCTYPE html>
<html>
    <head>
        <title>403</title>
        <meta http-equiv="content-type" content="text/html;charset=utf-8" />
        <link
            href="{{asset('css/style.css')}}"
            type="text/css"
            rel="stylesheet"
        />
    </head>
    <body>
        <h1>403</h1>
        <p>You are not authorised to view this page</p>
    </body>
</html>
```

-   Test this works.

## More to Investigate

There's a lot more to authentication and authorisation. Here some topics you might want to investigate for Assignment 2.

-   Policies
-   Registration (sign up)
-   Password resetting
-   Login throttling
