---
title: "Hosting an API in Firebase writen with Typescript and Nest"
datePublished: Thu Aug 24 2017 01:03:40 GMT+0000 (Coordinated Universal Time)
cuid: clf8ymi8p079rxnnvd07h0po9
slug: hosting-an-api-in-firebase-writen-with-typescript-and-nest
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1678841050417/9b0ea4b7-c926-4c80-bd87-a4d7cfd57c4f.png
tags: firebase, typescript, nestjs

---

---

This week I decided to test if Firebase is suited to a personal project and to do it I needed to discovery how to host an API writen in Typescript with [Nest](https://github.com/kamilmysliwiec/nest) framework and, UOU, it was not an easy task! Because of this, I decided to write down the success path and registry some problems that I had.

As a disclaimer, I know it's a hell of a specificy project and not so many people (if anyone) will build with a stack like this, but if you want you can speed this process with this [repository](https://github.com/caiorcferreira/nest-js-firebase), which I write while making this post as a starter repo. Now, even if you won't build something with this stack you can get some tips that I will leave at the end of this post and may help you bootstrapting a project with a different stack.

### Pre-requisites

You will need the following packages installed:

* Firebase Tools
    
* @google-cloud/functions-emulator
    
* WebPack
    
* Nodemon
    

##### *TIP 1*

I had seen a lot of people having problems in running the firebase emulator, which is used to run the project locally . I suggest to install in the following way:

1. If you already have firebase tools installed, uninstall it with `$ npm remove -g firebase-tools`.
    
2. Then, install the package @google-cloud/functions-emulator with `$ npm install -g @google-cloud/firebase-functions`.
    
3. Then, install firebase tools again: `$ npm install -g firebase-tools`.
    

### Building project scaffold

Create your project folder and enter it

```plaintext
$ mkdir nestjs-firebase
$ cd nestjs-firebase
```

Use the Firebase CLI to create the initial project

```plaintext
$ firebase init
```

When you run this command, you will be promped with a configuration menu, then, follow the steps bellow:

* Mark the Functions and Hosting features with space and press enter.
    
    ![Select the Firebase features we wiil need](https://cdn.hashnode.com/res/hashnode/image/upload/v1678841050417/9b0ea4b7-c926-4c80-bd87-a4d7cfd57c4f.png align="left")
    
* Set your Firebase project
    
    ![Set your Firebase project](https://cdn.hashnode.com/res/hashnode/image/upload/v1678841052507/eb8fcf00-2ff4-4a56-8c64-aabb391d1bbe.png align="left")
    
* Enter `n` to not install npm dependencies yet
    
    ![Enter n](https://cdn.hashnode.com/res/hashnode/image/upload/v1678841054514/d81d7cdc-0864-4161-9971-77f778d68a77.png align="left")
    
* Press enter to use the default hosting directory as public.
    
    ![Press enter](https://cdn.hashnode.com/res/hashnode/image/upload/v1678841057015/d06d04c9-f6c1-42d3-8711-e1a29c887a5a.png align="left")
    
* Enter `n` to disable single-app configuration
    
    ![Enter n again](https://cdn.hashnode.com/res/hashnode/image/upload/v1678841059246/775a3c3b-ba3f-4e37-8a94-51e6200dbf0e.png align="left")
    

Then, open your favorite code editor. I will be using Visual Studio Code. You will see the following folder structure:

![Initial project folder structure](https://cdn.hashnode.com/res/hashnode/image/upload/v1678841061954/cba9205d-ad0d-4bae-830d-b56c2082586a.png align="left")

Now enter functions folder with `$ cd functions` and update the package.json file inside it with the following.

```json
{
  "name": "nestjs-firebase",
  "description": "NestJS app with Cloud Functions for Firebase",
  "scripts": {
    "build": "webpack",
    "serve": "webpack && firebase serve --only functions,hosting",
    "serve:dev": "nodemon -w *.ts --exec npm run serve",
    "deploy": "webpack && firebase deploy --only functions,hosting"
  },
  "dependencies": {
    "firebase-admin": "~4.2.1",
    "firebase-functions": "^0.5.9",
    "@nestjs/common": "*",
    "@nestjs/core": "*",
    "@nestjs/microservices": "*",
    "@nestjs/testing": "*",
    "@nestjs/websockets": "*",
    "@types/express": "^4.0.37",
    "express": "^4.15.4",
    "redis": "^2.7.1",
    "reflect-metadata": "^0.1.10",
    "rxjs": "^5.4.0",
    "typescript": "^2.3.2"
  },
  "devDependencies": {
    "@types/node": "^7.0.5",
    "ts-loader": "^2.3.3",
    "webpack-node-externals": "^1.6.0"
  },
  "private": true
}
```

Then, run `$ npm install`. After, create the src folder `$ mkdir src` and delete *index.js*. Next, create *webpack.config.js* `$ > webpack.config.js` with this.

```javascript
// webpack.config.js
'use strict';

const nodeExternals = require('webpack-node-externals');

module.exports = {
    entry: './src/server.ts',
    output: {
        filename: 'index.js', // <-- Important
        libraryTarget: 'this' // <-- Important
    },
    target: 'node', // <-- Important
    module: {
        rules: [
            {
                test: /\.tsx?$/,
                loader: 'ts-loader',
                options: {
                    transpileOnly: true
                }
            }
        ]
    },
    resolve: {
        extensions: [ '.ts', '.tsx', '.js' ]
    },
    externals: [nodeExternals()] // <-- Important
};
```

Now, create the this structure inside src folder, with the following content.

![src folder structure](https://cdn.hashnode.com/res/hashnode/image/upload/v1678841064798/9e339b3c-526e-45d2-a212-9eae7b8d0b04.png align="left")

```typescript
// server.ts
import * as functions from 'firebase-functions';
import * as express from 'express';

import { NestFactory } from '@nestjs/core';
import { ApplicationModule } from './modules/app.module';

const server = express();
const app = NestFactory.create(ApplicationModule, server);

app.init();
exports.api = functions.https.onRequest(server);
```

```typescript
// app.module.ts
import { Module } from '@nestjs/common';

@Module({})
export class ApplicationModule {}
```

Finally, update firebase.json like this.

```json
{
  "hosting": {
    "public": "public",
    "rewrites": [{
      "source": "**",
      "function": "api"
    }],
    "ignore": [
      "firebase.json",
      "**/.*",
      "**/node_modules/**"
    ]
  }
}
```

And, if you want to use the root url in your API, you need to delete or rename *public/index.html*.

##### *TIP 2*

If you are on Linux (I don't know if this happens in others OS), you can have problems when running `$ firebase serve`. It is caused by permission problems and you can solve this in two ways:

1. Run the serve and deploy commands from firebase with sudo.
    
2. Change the owner of the firebase-tools folder in the global node\_modules to your user.
    

I recommend the second way because, otherwise, when the nodemon restart the server it will not kill the previous process, so every restart the firebase tool will see the port 5000 as used and will start to listen in another process and this will pile up in your machine.

### Conclusion

Now, you can develop your project and use run it with this commands:

* To build the app `$ npm run build`
    
* To run the server `$ npm run serve`
    
* To run the server for development `$ npm run serve:dev`
    
* To deploy it to Firebase `$ npm run deploy`
    

Wish I helped someone and, if you have questions or suggetions, please leave your comment.

Thank you!

### Fonts

[https://medium.com/netscape/firebase-cloud-functions-with-typescript-and-webpack-7781c882a05b](https://medium.com/netscape/firebase-cloud-functions-with-typescript-and-webpack-7781c882a05b)

[https://github.com/ultrasaurus/firebase-functions-typescript](https://github.com/ultrasaurus/firebase-functions-typescript)

[https://stackoverflow.com/questions/44871075/redirect-firebase-hosting-root-to-a-cloud-function-is-not-working](https://stackoverflow.com/questions/44871075/redirect-firebase-hosting-root-to-a-cloud-function-is-not-working)

### Tips

[Tip 1 - Firebase Emulator problem](#tip-1) [Tip 2 - Firebase serve commands problem](#tip-2)