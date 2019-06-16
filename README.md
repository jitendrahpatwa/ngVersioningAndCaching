# Angular Versioning and Production Build Caching Issue

## Versioning is important in every build of angular project, specially while we deploy to the production environment. We have many ways to do that. 
## To overcome versioning of angular build, in past when we made production build we face assets caching issues in environment, some time browser does't refresh the entire updated asset files(.css, .js, etc); to resolving this I found not the best way but a realistic way to achieve that. 
## In this story we are going to show how we make versioning build for angular production deploy and second, how we resolve caching issue for production build asset files.


---

# Step 1:
## Create angular application, by using 
```
ng new <project-name>
```

# Install replace-in-file npm package by command using,
```
npm i replace-in-file
```
uses the npmjs package reference from https://www.npmjs.com/package/replace-in-file

# Step 2:
## In your app add below line of code in your src/environments/environment.ts & src/environments/environment.prod.ts
```
export const environment = {
 …
 version: '0.0.0'
 …
};
```
# Step 3:
## In your root project directory add below two files, you can name with your own.
`1 replace.build.js`
`2 versioning.build.js`
# Step 4:
## Specify version in your package.json in version object. I've used current as same as version: '0.0.0'.
For above image we can identify which version is required in every build. We can see more in here https://docs.npmjs.com/cli/version
# Step 5:
## Add following code in replace.build.js,
```
var replaceFile = require('replace-in-file');
var package = require("./package.json");
var buildVersion = package.version;
const options = {
 files: 'src/environments/environment.prod.ts',
 from: /version: '(.*)'/g,
 to: "version: '"+ buildVersion + "'",
 allowEmptyPaths: false,
};
try {
 let changeFiles = replaceFile.sync(options);
 if (changeFiles == 0) {
 throw "Please make sure file have version";
 }
 console.log('Build version set: ' + buildVersion);
}
catch (error) {
 console.error('Error occurred:', error);
 throw error
}
```
# Step 6:
## Add following code in versioning.build.js
```
var fs = require('fs');
var path = require('path');
var replaceFile = require('replace-in-file');
var package = require("./package.json");
var angular = require("./angular.json");
var buildVersion = package.version;
var buildPath = '\\';
var defaultProject = angular.defaultProject;
var appendUrl = '?v=' + buildVersion;
const getNestedObject = (nestedObj, pathArr) => {
 return pathArr.reduce((obj, key) =>
 (obj && obj[key] !== 'undefined') ? obj[key] : undefined, nestedObj);
}
const relativePath = getNestedObject(angular, ['projects',defaultProject,'architect','build','options','outputPath']); // to identify relative build path when angular make build
buildPath += relativePath.replace(/[/]/g, '\\');
var indexPath = __dirname + buildPath + '/' + 'index.html';
console.log('Angular build path:', __dirname , buildPath);
console.log('Change by buildVersion:',buildVersion);
fs.readdir(__dirname + buildPath, (err, files) => {
 files.forEach(file => {
 if (file.match(/^(es2015-polyfills|main|polyfills|runtime|scripts|styles)+([a-z0–9.\-])*(js|css)$/g)) { // regex is identified by build files generated
 console.log('Current Filename:',file); 
 const currentPath = file;
 const changePath = file + appendUrl;
 changeIndex(currentPath, changePath); 
 }
 });
});
function changeIndex(currentfilename, changedfilename) {
const options = {
 files: indexPath,
 from: '"'+ currentfilename + '"',
 to: '"'+ changedfilename + '"',
 allowEmptyPaths: false,
 };
try {
 let changedFiles = replaceFile.sync(options);
 if (changedFiles == 0) {
 console.log("File updated failed");
 } else if (changedFiles[0].hasChanged === false) {
 console.log("File already updated");
 }
 console.log('Changed Filename:',changedfilename);
 }
 catch (error) {
 console.error('Error occurred:', error);
 throw error
 }
}
```
# Step 7:
## Add a script in package.json file which were build the project in versioning.
```
"scripts" : {
…
"build-prod-patch": "npm version patch && node ./replace.build.js && ng build - prod - base-href=./ && node ./versioning.build.js",
 "build-prod-no-patch": "node ./replace.build.js && ng build - prod - base-href=./ && node ./versioning.build.js"
…
}
```
## Note: Command npm version patch is update the project version and in package.json hence we can not change to the lowest one to make a production build. So before patch we can check version by command, 
```
>npm version
```
### Therefore, we have added two script in package.json scripts node. 

#### One, is for using node internal patching. The command build-prod-patch is uses to native node internal patching. Its auto update with there next version by patching. For example, current version is 0.0.0 then after patch it should be 0.0.1.
#### Another, is by manual versioning. The command build-prod-no-patch is just uses the version from package.json where we should decide which version is to give for project.
## And, with replace.build.js file we uses this scenario to append angular versioning using environment.ts file.
## The caching issue is resolved by versioning.build.js file it uses the index.html file and update the build script and styles resources written in index script tag and link tags. It will change the tags value by example, 
<script src="runtime-es2015.858f8dd898b75fe86926.js" type="module"></script>
to
<script src="runtime-es2015.858f8dd898b75fe86926.js?v=0.0.0" type="module"></script>.
## So when production build is made and deployed to the host, browser loads index.html file with update scripts file and styles resources.
## To see I uses node http-server to run the dist/* folder which is deployed angular project by command,
```
>http-server -p 91
```