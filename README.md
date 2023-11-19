# ed-laravel-multi-auth
By [article](https://medium.com/@emmwenda/build-a-multi-auth-login-system-with-laravel-51951dc40f8a)

So you’re building a web application like a real estate project that requires different types of users .eg agents, admin, users to login using the same login page but they should be redirect to different dashboards? Look no further.Dive withe me into this guide to see how to do exactly that using Laravel 10

Lets install a fresh Laravel project by running the following command

```
composer create-project --prefer-dist laravel/laravel realestate
```

Change your current directory to the project location

```
cd realestate
```

To make work easier, we will be using breeze package for authentication so run each of the following commands which install the package and publish it in addition to compiling your applications front end assets

```
composer require laravel/breeze --dev

php artisan breeze:install

npm install 

npm run dev
```

You will notice we have not migrated our database yet as we need to update our user’s migration file to add a role column which will be used in authentication. Open users migration file and add the following column

```
$table->enum('role',['admin','agent','user'])->default('user');
```

What the above line does is that it declares our user types to be 3 .i.e a user can be either an admin or an agent or a normal user. This can be changed to match your requirements and it sets the user type to ‘user’ if no role has been submitted during user registration.

Update your database information in **.env** file

```
DB_DATABASE=your-database-name
DB_USERNAME=your-database-user
DB_PASSWORD=your-database-password
```

Before we perform migration, we need to submit a few test data so we will create a users table seeder having the 3 different types of user

```
php artisan make:seeder UsersTableSeeder
```

Add the dummy data as indicated below according to your different user types

```
<?php

namespace Database\Seeders;

use Illuminate\Database\Console\Seeds\WithoutModelEvents;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Hash;

class UsersTableSeeder extends Seeder
{
    /**
     * Run the database seeds.
     */
    public function run(): void
    {
        DB::table('users')->insert([
            //admin
            [
                'name' =>  'Admin',
                'email' => 'admin@gmail.com',
                'password' => Hash::make('password'),
                'role' => 'admin',
            ],

            //agent
            [
                'name' =>  'Agent',
                'email' => 'agent@gmail.com',
                'password' => Hash::make('password'),
                'role' => 'agent',
            ],

            //user
            [
                'name' =>  'User',
                'email' => 'user@gmail.com',
                'password' => Hash::make('password'),
                'role' => 'user',
            ]
        ]);
    }
}
```

Update the database seeders file and add this line inside the run method

```
 $this->call(UsersTableSeeder::class);
```

You can now migrate your database and seed your data using the following command

```
php artisan migrate:fresh --seed
```

Start your server and navigate to your localhost:8000 and try to login using the above details you used in your seeder file. It should be logging in successfully.

This brings us to the next milestone. All the three user types are able to login, but they are being redirected to the same dashboard upon login. Let’s change that by creating new routes,controllers for both the admin and agent users

```
php artisan make:controller AdminController

php artisan make:controller AgentController
```

Create routes to their different dashboards in the web.php routes file

```
Route::get('/admin/dashboard',[AdminController::class,'dashboard')->name('admin.dashboard');

Route::get('/agent/dashboard',[AgentController::class,'dashboard')->name('agent.dashboard');
```

Update the AdminController.php to add method

```
public function dashboard(){
  return view('admin.dashboard');
}
```

Also, update the AgentController.php to add a similar method

```
public function dashboard(){
  return view('agent.dashboard');
}
```

Create the dashboard files for admin and agent insider the views folder (resources/views/admin/dashboard.blade.php) for admin and for agent (resources/views/agent/dashboard.blade.php). This can be having plain html content for the time being as below

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>
    <h1>Admin Dashboard</h1>
</body>
</html>
```

> Update the authenticated session controller in (app/Http/Controllers/Auth/AuthenticatedSessionController) to include the following redirection logic so that a user type is redirected to a specific dashboard upon login

```
public function store(LoginRequest $request): RedirectResponse
    {
        $request->authenticate();

        $request->session()->regenerate();
        
        $url = "";
        if($request->user()->role === "admin"){
            $url = "admin/dashboard";
        }elseif($request->user()->role === "agent"){
            $url = "agent/dashboard";
        }else{
            $url = "dashboard";  
        }

        return redirect()->intended($url);
    }
```

> As per now, upon login a user is redirected to their respective dashboard .eg /agent/dashboard if the user is an agent . But if you change and replace the above url with /admin/dashboard the agent will still be able to access the admin route without being an admin because our routes have not been protected. Lets do that below using a middleware that we will create called Role.

```
php artisan make:middleware Role
```

Register the middleware in Kernel.php (located in app/Http) under the middleware aliases array

```
'role' => \App\Http\Middleware\Role::class,
```

In the Role middleware file, update to include the following code: this will redirect a user to the dashboard URL if their role doesn't match the allowed roles for the URL they are trying to access

```
public function handle(Request $request, Closure $next, $role): Response
    {
        if($request->user()->role !== $role){
            return redirect('dashboard');
        }
        return $next($request);
    }
```

We will now update the routes file(routes/web.php) to enforce the middleware we have created in specific routes

```
//admin routes
Route::middleware(['auth','role:admin'])->group(function () {
    Route::get('/admin/dashboard',[AdminController::class,'dashboard'])->name('admin.dashboard');
});


//agent routes
Route::middleware(['auth','role:agent'])->group(function () {
    Route::get('/agent/dashboard',[AgentController::class,'dashboard'])->name('agent.dashboard');
});
```

If you login as an agent and try accessing the admin dashboard, you will be redirected to user’s dashboard URL instead of accessing the admin URL.(as you don’t have permissions)
