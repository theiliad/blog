---
layout: post
title: "Bundling as-fast-as-possible with Fusebox"
description: "Build and deploy a sample React app, bundled through Fusebox with Hot-Module-Replacement and serve through Express"
date: 2017-07-20 17:30
category: Technical guide
banner:
  text: ""
author:
  name: "Eliad Moosavi"
  url: "https://eMoosavi.com"
  mail: "iliadmoosavi@gmail.com"
  avatar: "https://en.gravatar.com/userimage/8929810/b2394344c3cabf234ecc96964c6072c0.jpeg"
design:
  image: https://avatars1.githubusercontent.com/u/23119087?v=4&s=100
  bg_color: "#1d79bf"
tags:
- bundling
- fusebox
- react
- node
- express
---
`# "What in the world is bundling?"
Bundling in a web app is a **recipe** you'd make to **group and compress** a set of files with the same type in order for your users' browsers to download less files.

Using bundling tools you can also make those files go through certain **pipes** and come out a different file. (e.g. for .scss files you can apply a Sass pipe to convert them into css, and bundle them into 1 final CSS file)

![Webpack](http://i.imgur.com/Keem1SS.jpg)

Bundling is an important step in making **scalable web-apps** these days, and **Webpack** has been very popular for that.

Today I'm going to introduce to you a new tool that bundles **faster than webpack2**, is **much simpler**, and offers **built-in typescript support**:
![FuseBox](http://fuse-box.org/static/img/logo.png)

# Meet Fusebox
Fusebox is very fast in bundling (& rebundling), thanks to its "aggressive module caching".

It is also very simple to make a bundle using Fusebox, take this as an example:
```javascript
// Import Fusebox
const { FuseBox } = require("fuse-box");

// Fusebox Configurations
const fuse = FuseBox.init({
    homeDir: "src",
    output: "dist/$name.js",
});

// Name of the bundle and what file it starts from
fuse.bundle("app")
    .instructions(`>index.ts`);

// Fire it up!
fuse.run();
```
Simple huh?

# Let's build a simple React app
![React](http://i.imgur.com/7KtKvNf.gif)
First things first you need to install some things
```sh
npm install --save react react-dom reflect-metadata
```
```sh
npm install --save-dev babel-core babel-plugin-transform-react-jsx babel-preset-es2015 babel-preset-latest chalk fuse-box typescript uglify-js
```
# What did I just install?

  - react
  - babel libraries to be able to transpile react .jsx files into javascript
  - fuse-box
  - uglify-js to minify our final bundle

We're going to have to build some files now. Here's an overview of the final folder structure:
![Folder Structure](http://i.imgur.com/PEki8LO.jpg)

Let's build the Fusebox configurations; make a file named `fuse.js` with the code below:
```js
const {
    FuseBox,
    SVGPlugin,
    CSSPlugin,
    BabelPlugin,
    QuantumPlugin,
    WebIndexPlugin,
    Sparky
} = require("fuse-box");

let fuse, app, vendor, isProduction;

// Sparky is the fusebox task manager http://fuse-box.org/page/sparky
Sparky.task("config", () => {
    fuse = new FuseBox({
        homeDir: "src/",
        sourceMaps: !isProduction,
        hash: isProduction,
        output: "dist/$name.js",
        plugins: [
            // BabelPlugin will take care of all the jsx files in React
            BabelPlugin({
                config: {
                    sourceMaps: true,
                    presets: ["es2015"],
                    plugins: [
                        ["transform-react-jsx"],
                    ],
                },
            }),

            // SVG & CSS plugins will make sure to add CSS & SVG files to your bundle when you import them
            SVGPlugin(), CSSPlugin(),

            // This plugin adds your bundles to the "src/index.html" file and store the new file in the "dist" folder
            WebIndexPlugin({
                template: "src/index.html",
                path: "../dist"
            }),

            // Quantum plugin is an optional optimizer made by Fusebox to make your bundle work faster
            isProduction && QuantumPlugin({
                removeExportsInterop: false,
                uglify: true
            })
        ]
    });

    // Bundle the vendor scripts (react etc.)
    vendor = fuse.bundle("vendor").instructions("~ index.jsx")

    // Bundle the app
    app = fuse.bundle("app").instructions("> [index.jsx]")
});

// This is the default task that gets called when you run "node fuse"
Sparky.task("default", ["clean", "config"], () => {
    fuse.dev();

    // Hot Module Replacement
    app.watch().hmr()
    
    return fuse.run();
});

// This task cleans the dist folder everytime there's a new bundle to be made
Sparky.task("clean", () => Sparky.src("dist/").clean("dist/"));
```

Now create a folder called `src` & make a file named `index.jsx`:
```js
import React      from 'react';
import ReactDOM   from 'react-dom';
import App        from './App.jsx';

ReactDOM.render(
  <App />, document.getElementById('root'));
```

This will hold our react app inside of it, and render it to our html page.

Now make a file named `app.jsx` inside `src`

```js
import * as React       from "react";
import { Component }    from 'react';

// Importing some assets
import './App.css';
import logo from './logo.svg';

class App extends Component {
    render() {
        return (
            <div className="App">
                <div className="App-header">
                    <img src={logo} className="App-logo" alt="logo" />
                    <h2>Welcome to React!</h2>
                </div>

                <p className="App-intro">
                    To get started, edit
                    <code> src/App.js </code>
                    and save to reload.
                </p>
            </div>
        );
    }
}

export default App;
```

This is our App component, that will be rendered by `index.jsx`.

Now this App component also imports 2 files, `App.css` & `logo.svg`, let's also make those.

`App.css` inside `src`:
```css
.App {
    text-align: center;
}

.App-logo {
    animation: App-logo-spin infinite 20s linear;
    height: 80px;
}

.App-header {
    background-color: #222;
    height: 150px;
    padding: 20px;
    color: white;
}

.App-intro {
    font-size: large;
}

@keyframes App-logo-spin {
    from {
        transform: rotate(0deg);
    }
    to {
        transform: rotate(360deg);
    }
}
```

`logo.svg` inside `src`:
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 841.9 595.3">
    <g fill="#61DAFB">
        <path d="M666.3 296.5c0-32.5-40.7-63.3-103.1-82.4 14.4-63.6 8-114.2-20.2-130.4-6.5-3.8-14.1-5.6-22.4-5.6v22.3c4.6 0 8.3.9 11.4 2.6 13.6 7.8 19.5 37.5 14.9 75.7-1.1 9.4-2.9 19.3-5.1 29.4-19.6-4.8-41-8.5-63.5-10.9-13.5-18.5-27.5-35.3-41.6-50 32.6-30.3 63.2-46.9 84-46.9V78c-27.5 0-63.5 19.6-99.9 53.6-36.4-33.8-72.4-53.2-99.9-53.2v22.3c20.7 0 51.4 16.5 84 46.6-14 14.7-28 31.4-41.3 49.9-22.6 2.4-44 6.1-63.6 11-2.3-10-4-19.7-5.2-29-4.7-38.2 1.1-67.9 14.6-75.8 3-1.8 6.9-2.6 11.5-2.6V78.5c-8.4 0-16 1.8-22.6 5.6-28.1 16.2-34.4 66.7-19.9 130.1-62.2 19.2-102.7 49.9-102.7 82.3 0 32.5 40.7 63.3 103.1 82.4-14.4 63.6-8 114.2 20.2 130.4 6.5 3.8 14.1 5.6 22.5 5.6 27.5 0 63.5-19.6 99.9-53.6 36.4 33.8 72.4 53.2 99.9 53.2 8.4 0 16-1.8 22.6-5.6 28.1-16.2 34.4-66.7 19.9-130.1 62-19.1 102.5-49.9 102.5-82.3zm-130.2-66.7c-3.7 12.9-8.3 26.2-13.5 39.5-4.1-8-8.4-16-13.1-24-4.6-8-9.5-15.8-14.4-23.4 14.2 2.1 27.9 4.7 41 7.9zm-45.8 106.5c-7.8 13.5-15.8 26.3-24.1 38.2-14.9 1.3-30 2-45.2 2-15.1 0-30.2-.7-45-1.9-8.3-11.9-16.4-24.6-24.2-38-7.6-13.1-14.5-26.4-20.8-39.8 6.2-13.4 13.2-26.8 20.7-39.9 7.8-13.5 15.8-26.3 24.1-38.2 14.9-1.3 30-2 45.2-2 15.1 0 30.2.7 45 1.9 8.3 11.9 16.4 24.6 24.2 38 7.6 13.1 14.5 26.4 20.8 39.8-6.3 13.4-13.2 26.8-20.7 39.9zm32.3-13c5.4 13.4 10 26.8 13.8 39.8-13.1 3.2-26.9 5.9-41.2 8 4.9-7.7 9.8-15.6 14.4-23.7 4.6-8 8.9-16.1 13-24.1zM421.2 430c-9.3-9.6-18.6-20.3-27.8-32 9 .4 18.2.7 27.5.7 9.4 0 18.7-.2 27.8-.7-9 11.7-18.3 22.4-27.5 32zm-74.4-58.9c-14.2-2.1-27.9-4.7-41-7.9 3.7-12.9 8.3-26.2 13.5-39.5 4.1 8 8.4 16 13.1 24 4.7 8 9.5 15.8 14.4 23.4zM420.7 163c9.3 9.6 18.6 20.3 27.8 32-9-.4-18.2-.7-27.5-.7-9.4 0-18.7.2-27.8.7 9-11.7 18.3-22.4 27.5-32zm-74 58.9c-4.9 7.7-9.8 15.6-14.4 23.7-4.6 8-8.9 16-13 24-5.4-13.4-10-26.8-13.8-39.8 13.1-3.1 26.9-5.8 41.2-7.9zm-90.5 125.2c-35.4-15.1-58.3-34.9-58.3-50.6 0-15.7 22.9-35.6 58.3-50.6 8.6-3.7 18-7 27.7-10.1 5.7 19.6 13.2 40 22.5 60.9-9.2 20.8-16.6 41.1-22.2 60.6-9.9-3.1-19.3-6.5-28-10.2zM310 490c-13.6-7.8-19.5-37.5-14.9-75.7 1.1-9.4 2.9-19.3 5.1-29.4 19.6 4.8 41 8.5 63.5 10.9 13.5 18.5 27.5 35.3 41.6 50-32.6 30.3-63.2 46.9-84 46.9-4.5-.1-8.3-1-11.3-2.7zm237.2-76.2c4.7 38.2-1.1 67.9-14.6 75.8-3 1.8-6.9 2.6-11.5 2.6-20.7 0-51.4-16.5-84-46.6 14-14.7 28-31.4 41.3-49.9 22.6-2.4 44-6.1 63.6-11 2.3 10.1 4.1 19.8 5.2 29.1zm38.5-66.7c-8.6 3.7-18 7-27.7 10.1-5.7-19.6-13.2-40-22.5-60.9 9.2-20.8 16.6-41.1 22.2-60.6 9.9 3.1 19.3 6.5 28.1 10.2 35.4 15.1 58.3 34.9 58.3 50.6-.1 15.7-23 35.6-58.4 50.6zM320.8 78.4z"/>
        <circle cx="420.9" cy="296.5" r="45.7"/>
        <path d="M520.5 78.1z"/>
    </g>
</svg>
```
Now we just need our `index.html` file to show our react app.

Now with Fusebox you can use a variable named `$bundles`, and what that will do is it will make you bundle files and automatically add replace them with `$bundles` in your html file.

`index.html` inside `src`:
``` html
<!DOCTYPE html>
<html>
<head>
    <title>React + Fusebox</title>
    <style>
        body
            margin: 0;
            padding: 0;
            font-family: sans-serif;
        }
    </style>
</head>
<body>
</body>
<div id="root"></div>
$bundles

</html>
```
# We're ready to make our bundle now
You can now run:
```sh
node fuse
```

to make a bundle out of this react app. The results will go inside a folder named `dist`.

# Let's make a simple server to show our app
Now we'll make a simple NodeJS (express) server to just serve our `index.html` file that's generated inside `dist`

Make a new folder named `server` and store this code inside `server.js`:
```js
const express         = require('express')
      , path          = require('path')
      , http          = require('http')
      , chalk         = require('chalk');

const app = express();

app.use('/dist', express.static(__dirname + '/../dist'));


app.get('*', (req, res) => {
  res.sendFile(path.resolve('dist/index.html'));
});

const port = process.env.PORT || '3000';

const httpServer = http.createServer(app);

httpServer.listen(port, () => {
  console.log(chalk.green(`Server running on http://localhost:${port}`));
});
```

Now from the root directory of the project, run:
```sh
node server/server.js
```
Now go to `localhost:3000` to see your app!

Here's also a [Github repo](https://github.com/theiliad/node-fusebox-react) with everything we just made!