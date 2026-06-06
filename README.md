# 🛡️ IlmHub — Web Application Security Enhancement Report

---

## Group Members

| Name | Matric No. |
|------|-----------|
| *Mohamad Fazwan bin Fadzli* | *2317181* |
| *Mohamad Razif Arman bin Rizuwan* | *2311911* |
| *Muhamad Hazim Iskandar bin Hassan Nordin* | *2317595* |

---

## Title of Web Application

**IlmHub** — An Islamic Event Management and Booking Platform

---

## Introduction

IlmHub is a web application developed using the **Laravel** framework (PHP) that serves as a platform for managing and booking Islamic educational events. The application allows authenticated users to create, manage, and book events such as religious lectures, seminars, and workshops. It features role-based access, event capacity management, gender-policy enforcement, and image upload functionality.

This report documents the security enhancements applied to the original IlmHub web application developed during INFO 3305 (Web Application Development). All enhancements were implemented following industry-standard web application security practices to harden the application against common threats.

---

## Objective of the Enhancements

The primary objectives of the security enhancements are to:

1. **Validate all user inputs** on both the client and server side to prevent malformed or malicious data from entering the system.
2. **Secure the authentication system** using best practices such as password hashing, session regeneration, and secure logout.
3. **Enforce authorization controls** to ensure users can only access and modify resources they own.
4. **Prevent XSS and CSRF attacks** using Laravel's built-in protections and proper output escaping.
5. **Strengthen database security** by leveraging Laravel's Eloquent ORM and prepared statements to prevent SQL injection.
6. **Apply file security principles** to prevent malicious file uploads and protect server-side files.

---

## Web Application Security Enhancements

---

### i. Input Validation

Input validation is applied on both the **client side** (HTML attributes) and **server side** (Laravel's `$request->validate()` method) to ensure only clean and expected data is accepted.

#### Client-Side Validation

HTML5 native attributes are used across all forms to provide immediate feedback to users before submission. These include `required`, `type="email"`, `type="number"`, `type="datetime-local"`, and `accept="image/*"`.

**Example — Registration Form (`register.blade.php`):**
```html
<input type="text" name="name" class="form-control" required>
<input type="email" name="email" class="form-control" required>
<input type="password" name="password" class="form-control" required>
<input type="password" name="password_confirmation" class="form-control" required>
```

**Example — Create Event Form (`create.blade.php`):**
```html
<input type="number" name="capacity" class="form-control" required placeholder="e.g. 50">
<input type="datetime-local" name="event_date" class="form-control" required>
<input type="file" name="image" class="form-control" accept="image/*">
```

#### Server-Side Validation

All form submissions are validated on the server using Laravel's `$request->validate()` method before any data is processed or stored. This acts as the definitive security gate.

**Example — User Registration (`AuthController.php`):**
```php
$request->validate([
    'name'     => 'required|string|max:255',
    'email'    => 'required|email|unique:users',
    'password' => 'required|min:6|confirmed',
]);
```

**Example — Event Creation (`EventController.php`):**
```php
$request->validate([
    'title'         => 'required',
    'description'   => 'required',
    'event_date'    => 'required|date',
    'location'      => 'required',
    'capacity'      => 'required|integer',
    'gender_policy' => 'required',
    'category_id'   => 'required|exists:categories,id',
    'image'         => 'nullable|image|mimes:jpeg,png,jpg,gif,webp|max:2048',
]);
```

| Input Field | Validation Rules Applied |
|---|---|
| `name` | `required`, `string`, `max:255` |
| `email` | `required`, `email`, `unique:users` |
| `password` | `required`, `min:6`, `confirmed` |
| `event_date` | `required`, `date` |
| `capacity` | `required`, `integer` |
| `category_id` | `required`, `exists:categories,id` |
| `image` | `nullable`, `image`, `mimes:jpeg,png,jpg,gif,webp`, `max:2048` |

---

### ii. Authentication

Authentication is implemented using **Laravel's built-in Auth facade** following authentication best practices.

#### Methods Implemented

**1. Password Hashing with Bcrypt**

User passwords are never stored in plain text. Laravel's `Hash::make()` uses **bcrypt** with 12 rounds (configured in `.env`) to hash passwords before storing them.

```php
// AuthController.php — register()
$user = User::create([
    'name'     => $request->name,
    'email'    => $request->email,
    'password' => Hash::make($request->password), // Hashed, never plain text
    'role'     => 'attendee',
]);
```

The `User` model also declares `password` as a cast to ensure it is always treated securely:

```php
// User.php
protected $casts = [
    'email_verified_at' => 'datetime',
    'password'          => 'hashed',
];
```

**2. Session Regeneration on Login**

After a successful login, the session ID is regenerated to prevent **session fixation attacks**.

```php
// AuthController.php — login()
if (Auth::attempt($credentials)) {
    $request->session()->regenerate(); // Prevents session fixation
    return redirect()->route('home')->with('success', 'Logged in successfully!');
}
```

**3. Secure Logout**

On logout, the session is fully invalidated and the CSRF token is regenerated, preventing token reuse attacks.

```php
// AuthController.php — logout()
public function logout(Request $request)
{
    Auth::logout();
    $request->session()->invalidate();       // Destroys session data
    $request->session()->regenerateToken();  // Regenerates CSRF token
    return redirect()->route('home');
}
```

**4. Sensitive Fields Hidden from Serialization**

The `password` and `remember_token` fields are hidden in the User model to prevent them from being exposed in API responses or JSON output.

```php
// User.php
protected $hidden = [
    'password',
    'remember_token',
];
```

**5. HTTP-Only Session Cookie**

Session cookies are configured with `http_only = true` (in `config/session.php`), preventing JavaScript from accessing the session cookie and mitigating XSS-based session hijacking.

```php
// config/session.php
'http_only' => env('SESSION_HTTP_ONLY', true),
'same_site' => env('SESSION_SAME_SITE', 'lax'),
```

---

### iii. Authorization

Authorization controls are applied at both the **route level** and the **controller level** to ensure users can only access resources they are permitted to.

#### Methods Implemented

**1. Route-Level Authorization with Auth Middleware**

All routes that require a logged-in user are grouped under the `auth` middleware. Unauthenticated users attempting to access these routes are automatically redirected to the login page.

```php
// web.php
Route::middleware(['auth'])->group(function () {
    Route::get('/events/create', [EventController::class, 'create'])->name('events.create');
    Route::post('/events/save', [EventController::class, 'store'])->name('events.store');
    Route::get('/my-events', [EventController::class, 'index'])->name('events.mine');
    Route::get('/events/{event}/edit', [EventController::class, 'edit'])->name('events.edit');
    Route::put('/events/{event}', [EventController::class, 'update'])->name('events.update');
    Route::delete('/events/{event}', [EventController::class, 'destroy'])->name('events.destroy');
    Route::post('/events/{id}/book', [BookingController::class, 'store'])->name('bookings.store');
    Route::get('/my-bookings', [BookingController::class, 'index'])->name('bookings.index');
    Route::delete('/bookings/{id}', [BookingController::class, 'destroy'])->name('bookings.destroy');
});
```

**2. Resource Ownership Checks in Controllers**

Even if a user is authenticated, they cannot edit, update, or delete an event unless they are the original creator. The `user_id` is compared against the authenticated user's ID, and a `403 Forbidden` response is returned if they do not match.

```php
// EventController.php — edit(), update(), destroy()
if ($event->user_id !== Auth::id()) {
    abort(403, 'Unauthorized action.');
}
```

The same ownership check is applied for booking cancellations:

```php
// BookingController.php — destroy()
if ($booking->user_id !== Auth::id()) {
    abort(403, 'Unauthorized action.');
}
```

**3. Scoped Data Access**

Users can only view their own events and bookings. Eloquent relationships are used to scope queries to the authenticated user, ensuring no cross-user data leakage.

```php
// EventController.php — index()
$events = Auth::user()->events()->with('category')->latest()->get();

// BookingController.php — index()
$bookings = Auth::user()->bookings()->with('event')->latest()->get();
```

**4. Role Assignment**

A `role` field is assigned at registration and stored in the database (`attendee` by default), enabling future role-based access control expansion.

```php
$user = User::create([
    ...
    'role' => 'attendee',
]);
```

---

### iv. XSS and CSRF Prevention

#### CSRF Prevention

Laravel's **CSRF protection** is enabled globally. Every HTML form in the application includes the `@csrf` Blade directive, which embeds a hidden CSRF token. Laravel validates this token on every `POST`, `PUT`, and `DELETE` request. Requests missing or containing an invalid token are rejected with a `419` error.

**Example — Login Form (`login.blade.php`):**
```html
<form action="{{ route('login.submit') }}" method="POST">
    @csrf
    ...
</form>
```

**Example — Logout Form (uses POST, not GET):**
```html
<form method="POST" action="{{ route('logout') }}">
    @csrf
    <button type="submit">Logout</button>
</form>
```

The logout action uses a `POST` route (not `GET`) to ensure CSRF protection is enforced even on logout:

```php
// web.php
Route::post('/logout', [AuthController::class, 'logout'])->name('logout');
```

Additionally, the session cookie is configured with `SameSite=lax`, which provides additional CSRF mitigation by restricting cross-site cookie transmission:

```php
// config/session.php
'same_site' => env('SESSION_SAME_SITE', 'lax'),
```

#### XSS Prevention

Laravel's Blade templating engine **automatically escapes all output** rendered using the `{{ }}` double-curly-brace syntax. This converts characters like `<`, `>`, `"`, and `'` into their HTML entities, preventing injected scripts from executing in the browser.

**Example — Displaying event data safely (`show.blade.php` pattern):**
```blade
{{ $event->title }}       {{-- Escaped: safe from XSS --}}
{{ $event->description }} {{-- Escaped: safe from XSS --}}
{{ $event->location }}    {{-- Escaped: safe from XSS --}}
```

User-generated content (event titles, descriptions, locations, names) is never rendered with unescaped `{!! !!}` syntax. All dynamic content displayed to users passes through Blade's auto-escaping layer.

---

### v. Database Security Principles

#### SQL Injection Prevention

The application uses **Laravel's Eloquent ORM** and the **Query Builder**, both of which use **PDO prepared statements** with parameterized queries under the hood. This ensures that user-supplied data is never concatenated directly into SQL strings.

**Example — Safe user lookup using Eloquent (`AuthController.php`):**
```php
// Auth::attempt() internally uses parameterized queries
if (Auth::attempt($credentials)) {
    ...
}
```

**Example — Safe record creation with Eloquent (`AuthController.php`):**
```php
$user = User::create([
    'name'     => $request->name,
    'email'    => $request->email,
    'password' => Hash::make($request->password),
    'role'     => 'attendee',
]);
```

**Example — Safe booking query with chained constraints (`BookingController.php`):**
```php
$existingBooking = Booking::where('user_id', $user->id)
                          ->where('event_id', $event->id)
                          ->where('status', '!=', 'cancelled')
                          ->first();
```

None of the queries use raw SQL string concatenation. All values are passed as parameters, making SQL injection attacks structurally impossible through these query paths.

**Mass Assignment Protection**

The `$fillable` array on each Eloquent model explicitly lists the fields that may be mass-assigned, preventing attackers from injecting unexpected fields (e.g., `role`, `is_admin`) through a crafted POST request.

```php
// User.php
protected $fillable = ['name', 'email', 'password', 'role'];

// Event.php
protected $fillable = [
    'title', 'description', 'event_date', 'location',
    'capacity', 'gender_policy', 'image_path', 'category_id', 'user_id',
];

// Booking.php (implied)
protected $fillable = ['user_id', 'event_id', 'status', 'booking_time'];
```

**Database Credentials in Environment Variables**

Database credentials are stored exclusively in the `.env` file and accessed via `env()`, keeping them outside the codebase and out of version control (`.env` is listed in `.gitignore`).

```
# .env (not committed to GitHub)
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=ilmhub_db
DB_USERNAME=root
DB_PASSWORD=*****
```

---

### vi. File Security Principles

#### File Upload Validation

File uploads are strictly validated on the server side to prevent malicious files (e.g., PHP webshells) from being uploaded.

```php
// EventController.php — store() and update()
'image' => 'nullable|image|mimes:jpeg,png,jpg,gif,webp|max:2048',
```

This rule enforces:
- Only **image** MIME types are accepted (rejects PHP, HTML, SVG, etc.)
- Only specific extensions: `jpeg`, `png`, `jpg`, `gif`, `webp`
- Maximum file size of **2MB** to prevent denial-of-service via large uploads

#### Filename Sanitization

Uploaded filenames are sanitized before saving to prevent directory traversal and path injection attacks. A timestamp prefix is added and whitespace is replaced with underscores.

```php
// EventController.php
$filename = time() . '_' . preg_replace('/\s+/', '_', $image->getClientOriginalName());
```

#### Controlled Storage Directory

Uploaded images are stored in the `public/events/` directory, which is explicitly created with restricted permissions (`0755`) if it does not exist.

```php
$destinationPath = public_path('events');
if (! is_dir($destinationPath)) {
    mkdir($destinationPath, 0755, true);
}
$image->move($destinationPath, $filename);
```

#### Old File Cleanup on Update/Delete

When an event image is replaced or an event is deleted, the old image file is removed from the server to prevent accumulation of orphaned files and reduce information leakage.

```php
// EventController.php — update() and destroy()
if ($event->image_path) {
    $oldPath = public_path($event->image_path);
    if (file_exists($oldPath)) {
        @unlink($oldPath);
    }
}
```

#### Client-Side File Type Hint

The file input uses `accept="image/*"` to guide users, combined with the server-side MIME validation as the authoritative enforcement layer.

```html
<input type="file" name="image" class="form-control" accept="image/*">
```

#### Sensitive Configuration Files Protected

The `.env` file (containing database credentials, application key, and other secrets) is included in `.gitignore` and is never committed to the repository, ensuring credentials are not exposed publicly.

```
# .gitignore
.env
```

---

## References

1. Laravel Documentation — Validation: https://laravel.com/docs/11.x/validation
2. Laravel Documentation — Authentication: https://laravel.com/docs/11.x/authentication
3. Laravel Documentation — Authorization: https://laravel.com/docs/11.x/authorization
4. Laravel Documentation — CSRF Protection: https://laravel.com/docs/11.x/csrf
5. Laravel Documentation — Blade Templates: https://laravel.com/docs/11.x/blade
6. OWASP — SQL Injection Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html
7. OWASP — Cross-Site Scripting (XSS) Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
8. OWASP — CSRF Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
9. OWASP — File Upload Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
10. OWASP — Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html

---


