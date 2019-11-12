---
layout: post-kayoung
title: "Deploy Create-React-App using Azure"
date: 2019-11-12
categories:
  - Azure
  - React
description:
image: https://picsum.photos/2000/1200?image=1025
image-sm: https://picsum.photos/500/300?image=1025
---
This blog will talk about how to deploy an create-react-app to Azure using Visual Studio Code.

More about create-react-app: <a href="https://github.com/facebook/create-react-app">GitHub Link</a>.

<h4>To try this blog, you must have:</h4>
<ul>
  <li>A Microsoft Azure account.</li>
  <li>Create-react-app</li>
  <li>Visual Studio Code</li>
</ul>

<h3>Let's get started</h3>
<h4>Make some changes to the index.js file</h4>

In folder public, find `index.js`, and add the following code into the file. Without the following snippet, Azure will not recognise the "entry point" of the application, thus it will not show your content correctly.

```javascript
var express = require('express');
var server = express();
var options = {
  index: 'index.html'
};
server.use('/', express.static('/home/site/wwwroot', options));
server.listen(process.env.PORT);
```

<h4>Build the application</h4>

Open your create-react-app in VSCode.
In the terminal (Terminal -> New Terminal), build the application using `npm run build` or `yarn build`. This command builds the app for production to the build folder.

<h4>Deploying the web app</h4>

In VSCode, go to View -> Extensions, or `Shift + Command + X by default`.

Sign in to your Azure account.

Then, search for Azure App Service and install
<figure>
  <img src="../assets/image/2019-11-12/vscode_1.jpeg" alt="VSextension"/>
</figure>

Or click here to install: 
<a href="vscode:extension/ms-azuretools.vscode-azureappservice">Install the Azure App Service extension</a>

Right click your subscription, choose the operating system that you want to deploy to. If you are targeting Linux, choose `Create New Web App...`, if targeting Windows, choose `Create New Web App... (Advanced)`.
<figure>
  <img src="../assets/image/2019-11-12/vscode_2.jpeg" alt="Subscription"/>
</figure>

Type a globally unique name for your Web App and press ENTER. Valid characters for an app name are 'a-z', '0-9', and '-'.

If targeting Linux, select a Node.js version when prompted. An LTS version is recommended.

If targeting Windows using the Advanced option, follow the additional prompts:
<ol>
<li>Select Create a new resource group, then enter a name for the resource group.</li>
<li>Select Windows for the operating system.</li>
<li>Select an existing App Service Plan or create a new one. You can select a pricing tier when creating a new plan.</li>
<li>Choose Skip for now when prompted about Application Insights.</li>
<li>Choose a region near you or near resources you wish to access.</li>
</ol>

Select `Yes` when prompted to update your configuration to run npm install on the target server. Your app is then deployed.
<figure>
  <img src="../assets/image/2019-11-12/vscode_3.png" alt="Subscription"/>
</figure>

When the deployment starts, you're prompted to update your workspace so that later deployments will automatically target the same App Service Web App. Choose `Yes` to ensure your changes are deployed to the correct app.
<figure>
  <img src="../assets/image/2019-11-12/vscode_4.png" alt="Subscription"/>
</figure>

**If VSCode is ever asking which folder to deploy, choose `myApp/build`.**

<h4>Voila</h4>

Congrats! Your `create-react-app` has been successfully deployed!

<figure>
  <img src="../assets/image/2019-11-12/vscode_5.png" alt="Subscription"/>
</figure>

View your application online by clicking `Browse Website`.

<h4>Deploy again</h4>
Say, you have made some changes to your application and want to deploy the latest changes.

In the VSCode terminal,  `npm run build` or `yarn build`.

After building the application, go to your Azure Web App Extension, find your subscription, find your wab application, right click and you will see 