# Web Fundamentals Room MiniCTF

This is a great bite-sized room to get more comfortable with how the basics of how the web works. I personally really enjoyed this room. Before you look at the solutions below, be sure that you have attempted the challenges yourself. 

See the room [here.](https://tryhackme.com/room/webfundamentals)

## Solutions below

Before you can start the MiniCTF, remember to deploy the machine. 

### 1. What's the GET flag? 

You can easily solve this challenge with the [curl command](https://curl.haxx.se/).

To make a GET request to a url with curl follows this format:

```
curl your-url-here
```

After you deploy the machine, the following command will net you the flag:

```
curl http://your-machine-ip-address-here/ctf/get
```

### 2. What's the POST flag?

To make a POST request with curl, you need to add the -d flag (think d for data) when you use curl. Think of "flags" for commands as checkboxes you would tick for the command to do something specific. A post request with curl and "somedata" you're sending follows this format: 

```
curl -d "somedata" your-url-here
```

Knowing this, then, the following command will net you the flag:

```
curl -d "flag_please" http://your-machine-ip-address-here/ctf/post
```

### 3. What's the "Get a cookie" flag?

In your browser's address bar go the following url:

```
your-machine-ip-address-here/ctf/getcookie
```

Open up the developer tools (getting here varies slightly from browser to browser), go the JavaScript console, and type the following:

```
document.cookie
```

This shows you all of the cookies stored in a webpage. In this case, the only cookie stored in the visited webpage is the flag!

### 4. What's the "Set a cookie" flag? 

In your browser's address bar go to the following url: 

```
your-machine-ip-address-here/ctf/sendcookie
```

Open up the developer tools and type the following:

```
document.cookie="flagpls=flagpls"
```

This sets a cookie with the name "flagpls" and the value "flagpls". 

Refresh the page, and the flag will appear. 

Thank you for reading my writeup, and happy learning!

Last updated: 4.16.2020

