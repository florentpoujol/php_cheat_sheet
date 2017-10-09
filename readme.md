# PHP Cheat Sheet

Also see [PHP The Right Way](http://www.phptherightway.com/)


## Performance

### Session

Set `session.save_handler` to something else than `files`. Use memory cache like Redis or Memcached instead of files (also good for security).





## Security

[PHP Manual on security](https://secure.php.net/manual/en/security.php)

### Configuration

Always have the latest version of all technologies involved.

`open_basedir` sets the folder where PHP has the right to use the file system.
Limit it as much as possible (ideally the root of your project) but don't forget special cases where PHP needs to fiddle with other folders (like `/tmp` for file upload).

```
open_basedir = /tmp:/var/www/mywebsite
```

Set `allow_url_include` to `off` as this prevent Remote File Include attacks.

If possible, set `allow_url_fopen` to `off`, but it is frequently needed to have it activated.

Set `expose_php` to `0` to not display the PHP version in request's headers. Make sure your web server doesn't do that on its own.





### User input

Always thoroughly check received data.

Check for type, use strict comparison, explicitly cast when expecting non-string types (int, bool or array).

Then check for value :
- check that a string both does not contains any unwanted character and that it contains all of the required chars
- check that the value (or number of chars) is not out of bounds
- if possible check the value against a set of possible values

Users can modify URLs and forms structure, so request can comes with fewer or more data than expected, so do not have a simple loop through the data.
Always explicitly process the expected data, not more, not less.

Use the Filter extension for checks and/or sanitation, or regexes, or a full-fledge validation system.  
Always do theses checks server-side, and at the last moment (to make sure the data is not modified between its check and its usage),

When you receive text to be later displayed, ideally prevent it to be submitted at all with unwanted characters for instance, and/or use `strip_tags()` on it before saving it in databse.  
But always at least escape it with `htmlentities()` when been displayed.

When the user should pass a path, be extra carefull to prevent the user accessing unwanted files: 
- remove leading `/` so that the path is relative, 
- remove in-between `../` to prevent going up in the hierarchy
- and remove trailing null-bytes (`\0`) to make sure the path ends with something you specify.

When uploading files:
- only allow for specific mime-types
- check the actual mime-type of a file, do not suppose based on the extension. Ie:
```
$finfo = new finfo(FILEINFO_MIME_TYPE);
$mimeType = $finfo->file($_FILES['upload_file']['tmp_name']);  

$allowedMimeTypes = ["image/jpeg", "image/png", "application/pdf", "application/zip"];
$isValidMimeType = in_array($mimeType, $allowedMimeTypes, true);
```
- rename the file on the server

Even when the input have been filtered, always query the DB with prepared statements.  

If some user input are part of a shell command, be extra careful and use `escapeshellcmd()` and `escapeshellarg()`.




### Sessions and Cookies

Do not thrust cookies.  
Users can easilly alter their cookies or have them stolen.

Do not store anything critical (ideally anything at all by the session id) in cookies, use sessions or a per-session server-side storage instead.
If your need per-user cross-section storage, prefer the database to the cookie when possible, even for small data.

Hashing or encryting the content of a cookie is usually useless, if the mere presence of the cookie on a user's computer is enough to produce side effect on the website (like authentication).

Do not make that the mere presence of the cookie allow someone to access critical content.

When logging off someone or destroying a session, make sure to also destroy the corresponding cookie(s).

If saving session in files, set `session.save_path` to something appropriate. Session are by default stored in files. Make sure the folder they are saved in is not readable by anyone (sometimes it's `/tmp`).
For performance you can also save the session in something else than files, like Redis. See above, the Performance section.

If you have a SSL certificate, set `session.cookie_secure` to `1` so that cookie can oly be transmitted over HTTPS.  
Can also be set on a per-cookie basis with the `secure` param of the `setcookie()` function.

Set `session.cookie_httponly` to `1`. It makes cookies only accessible through HTTP protocol but not other languages like JS, so it limits XSS attacks.  
Can also be set on a per-cookie basis with the `httponly` param of the `setcookie()` function.

Make sure `session.use_strict_mode` is set to `0`. Session id will be regenerated if a received session id is unknown. Helps with Session Fixation.

Make sure `session.use_trans_sid` is set to `0`. This deactivate the passing of the session id in all URL when the user has deactivated cookies.

Make sure `session.use_only_cookies` is set to `1`. PHP will ignore any session id passed in a URL.

Destroy and regenerate a session when a user log in. It helps with Session Fixation.


### Cryptography and hash

Use `password_hash()` and `password_verify()` for passwords.

Do not use `md5` or `sha1` for hashing important data.

Use `crypt()` or `hash()` for alternative hashing algorithms.

For encryption, only use the `openSSL` or `libsodium` extentions, uninstall `mcrypt`. 

### Misc

Do not let a "phpinfo" file lying around, especially with a name like `info.php`.

Have as much files as possible outside of the webserver document root. Typically you only need your front controller (index.php) and the static files (.css, .js, imgs...).

In addition to the rule above, configure the webserver so that it just can't serve some type of file or whole directories but the webroot.

Do not allow any (server) user to write in any folders they shouldn't. The files of your application do not need to be written by the user executing the webserver or PHP.

Use URL rewriting to hide the fact that your front controller is `index.php`.

Always validate permission for every actions (access, modification), do not suppose that a user is allowed to do something because he was able to do it.

Never display errors in production, but log them (and check and rotate the logs regularly).
```
display_errors = 0
display_startup_errors = 0
error_reporting = E_ALL ; or whatever combinaison you want
log_errors = /path/to/php_errors.log
```

Do not commit password or API keys in version control.  
Search your repo for such things. If some are discovered, check if they are still valid (and change them if yes).



### Database

Query the DB with a read-only user if there is no need to write.

The users that read and write probably don't need to be able to update the structure.

Prefer using a query builder or even an ORM instead of writing your query yourself.

Always use prepared queries, for security (and performance) to prevent SQL injection, which doesn't prevent you to sanitize the input data.

Prefer using PDO instead of mysqli or other native drivers. 


### CSRF

Cross Site Request Forgery is when an attacker sends a request to a website _in the context_ of an authenticated user. The request is accepted by the site like if it was sent by that user, from a link or a form on that website.  
The solution is to generate a cryptographically generated unique token, and link it to the user's session and to the request (in the URL for GET or as hidden field for POST).  
For improved UX, have a system that allows a single user to have several tokens (so that it can perform other actions while having a form open, for instance).  
Upon receiving a request, the site expects a token with the request and compare it with the token(s) stored in the user's session.  
Request is not accepted if the token don't match, or is accepted and the token is destroyed.

Generally speaking, do not use GET request to do anything other than read data.  
Use POST (or other HTTP methods) to do everything else, it make an attackers life a little harder.


### XSS

Typically when an attacker manage to execute malicious code (HTML, CSS or JS) on your website, typically by injecting it through a form and having it display on a page.


### SQL Injection

When some SQL is injected by a malicious user into a request, in order to perform unwanted request.
Use PDO's prepared statements.


### Session thief

When an attacker manage to steal a valid session id, so he can "be authenticated" as the regular user.
Usually this is done through stealing the session cookie via XSS, or managing to get an old URL with the session id, or more rarely to guess the session id.

This can be prevented by storing information about the user along the session like IP, user agent, locale.
If a user present the session id but the other infos don't match, destroy the session.


### Session Fixation

When an attacker manage to give a session id to it's victim. The idea is that it will wait for the user to authenticate with that session id.
Effect is negated by either regenerating session ids when they are unknown (not generated by the server, or old session), or by creating an all-new session when a user authenticate (at least regenerate the id).
