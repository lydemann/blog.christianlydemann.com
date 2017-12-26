---
title: "Angular Production Build"
date: 2017-12-26T17:45:58+01:00
draft: false
---

This is an overview of how to prepare an Angular app for production. There will be future posts going more in depth with the various steps described here. Note: Angular CLI is fortunately doing a lot of this out of the box, but it’s still valuable to know what it is doing in case you need to set this up without Angular CLI.
When preparing an Angular app for production it is important to optimize it in the various areas:

* Performance
* Bundle size  
* Security
 
Bundle performance, bundle size and security can have tradeoffs with each other. Finding the right balance depends on the requirements/needs of the application.

 
#### What is AoT?
AoT or Ahead of Time compilation is, as opposite to just in time compilation (jit), when the Angular application is compiled during build of the bundle instead of during runtime. Having the angular application build when served to the browser improves load time, because the templates are precompiled, and can reduce the application bundle size because the Angular compiler is not needed (but aot can actually make the bundle bigger if there are lots of templates that will be aot compiled).
 
#### Benefits of AoT:

##### No need for include Angular compiler
When the Angular app is ahead of time compiled, the Angular compiler is not needed in the bundle and thus is not shipped. These gives a smaller bundle size.

##### More performant as ng does not need to parse and create views on runtime
When the client loading the Angular application the templates are already compiled which will speed up the load time of the application.

##### More secure thus not using the eval to compile the app at runtime
The application is more secure because less code is being eval'ed at runtime using javascripts *eval* command and therefore less places for scripting injection attacks.
 
#### Setting up AoT in Webpack:
AoT can be set up in webpack using [@ngtools/webpack]( https://github.com/angular/angular-cli/tree/master/packages/%40ngtools/webpack) AotPlugin, which is part os Angular CLI:
```Javascript
      new AotPlugin({
            "mainPath": "main.ts",
            "hostReplacementPaths": {
              "environments\\environment.ts": !isProd ? "environments\\environment.ts" : "environments\\environment.prod.ts"
            },
            "exclude": [],
            "tsConfigPath": "src\\tsconfig.app.json",
            "skipCodeGeneration": !isProd
          })

```
The skipCodeGeneration option will determine whether the js bundle should be Aot compiled.

#### Bundling:
For minifying the amount of files for the client to retrieve, and thus more network requests, the js and css files should be bundled. 

##### Tree shaking

Not all code in the different js libraries are being used. Removing unused code will reduce the bundle size. A technique for removing unused code is tree shaking in which a bundler only includes used code in a bundle and leaves everything else out of the bundle. Two popular tools for tree shaking are [Webpack](https://webpack.github.io/) and [Rollup](https://rollupjs.org/). In short Webpack is, in my opinion, more suited for application bundling because of its plugin support for ts, js and scss/css transplication and bundling. Rollup is more used for Node module bundling. I am in my [Angular starter repo]( https://github.com/lydemann/Angular-4-Webpack-Starter) using Webpack for this reason and Angular CLI is also using Webpack.

##### Separation of bundles

It is beneficial to separate application, vendor and polyfills code in the bundles for easier error tracing and segregation of responsibilities in the bundles: 

1. The app bundle should contain only the application code 
2. The vendor is third party modules, eg. bootstrap 
3. The Polyfill bundle are files for actually running the app.\\

Im using CommonsChunkPlugin to separate the bundles by simply registrering in the plugins section of a webpack config:

```javascript
plugins: [
    new webpack.optimize.CommonsChunkPlugin({
        name: ['app', 'vendor', 'polyfills']
    }),
    ...
]
```

#### Minification:
Production files should always be minified to reduce bundle size.

##### Minifying css:
Css should be minified for reducing the total bundle size a lot. When bundling css it is recommended to inline css inside the /app folder and use extractTextPlugin to extract css outside the /app folder, eg. Css from NODE_MODULES. Use postcss-loader to minifying css. See my webpack  [common](https://github.com/lydemann/Angular-4-Webpack-Starter/blob/master/config/webpack.common.js) file for an example.
 
##### Minifying js:
For minifying js (and transpiling ts) I am using [@ngtools/webpack]( https://github.com/angular/angular-cli/tree/master/packages/%40ngtools/webpack) from the Angular CLI webpack tools. This tool offers great support for transpilation of typescript files, minification as well as code optimizations as mentioned before. 
 
#### Gzip (if not handled by server):
Servers like ASP.NET have support for gzip'ing and caching files requested frequently which is preferable. If the server doesn't support this the bundle size can be reduced a lot if the bundle is gzip'ed as part of the production build process. For gzipping the bundles it can be done as part of the webpack build process with [this](https://github.com/webpack-contrib/compression-webpack-plugin).

### Angular production build checklist:
- [ ] Aot compiled app
- [ ] App is tree shaken with Webpack 
- [ ] Code split in app, vendor and polyfills bundles 
- [ ] Bundles are minified
- [ ] Bundles are compressed with gzip (preferable server side)

### Conclusion 
This was my first post on the blog. It gave a brief overview of the tasks to get an Angular app ready for production. This is based on my experience, so feel free to try other tools than the ones mentioned in this post, if you like.
