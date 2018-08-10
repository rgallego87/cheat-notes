# New MEAN Project

## Starter code

We are about to create a new project, where we'll be using Express.js for the BackEnd part (storing the API) and Angular.io for the FrontEnd. In order to achieve this, we need to combine 2 different projects to manage both the different parts of the application. 



In our main project folder, we will create two separated parts: 

- Angular.io project - in the terminal, route  `/project-folder`:

  `$ ng new front-end-name	Ex: 'client'`

  

- Express.js project - in the terminal, route  `/project-folder`:

  `$ express --git back-end-name	Ex.: 'server'`

  

## Backend part

*NOTE: Express would generate the files in ECM5. Change variable declaration to ECM6!



### Set up for the backend

First of all, remember to install all the packages included in the `package.json`

+ In terminal, route: `/project-folder/server:`

  `$ npm install`



#### Delete views engine setup

Now we can start modifiying our default `Express.js` installation. As we won't need any rendering service from the Backend part of our aplication, we can delete the default `views engine setup`:

+ in `app.js`

```javascript

// view engine setup
app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'ejs');
```



#### Replace error handle

As we're creating our own API to manage the connexions with the FrontEnd part of our app, we need to change the way the errors are handleded in the Backend **(status + json response!)**. In order to do this, we can Replace the error handle with the following code: 

+ in `app.js`

```javascript

// catch 404 and forward to error handler
app.use((req, res, next) => {
  res.status(404).json({code: 'not found'});
});

app.use((err, req, res, next) => {
  // always log the error
  console.error('ERROR', req.method, req.path, err);

  // only render if the error ocurred before sending the response
  if (!res.headersSent) {
    res.status(500).json({code: 'unexpected'});
  }
});
```



#### Install `nodemon`

We will now install `nodemon` so we won't need to reset the server every time we make changes: 

+ In terminal inside: `project-folder/server:`

  ```$ npm install -D nodemon```



To run it, remember to add the following code to be able to run `npm run dev` in the terminal: 

+ in ```package.json```

  ```"dev": "nodemon  ./bin/www"```

 

#### Install `mongoose`

We will need to install `mongoose` to manage our conexions with the database: 

+ In terminal inside: `project-folder/server:`

  ```$ npm install --save mongoose```



### Connect with the database 

We are going to create an external file in our `project-folder` so we can keep our code cleaner. Let's create a file in our `project-folder/server` called `database.js.`

+ in `database.js`

```javascript

const mongoose = require ('mongoose')
require('dotenv').config();
mongoose.connect(process.env.MONGODB_URI, {
  keepAlive: true,
  reconnectTries: Number.MAX_VALUE,
    userNewUrlParser: true
})
.then(()=> {
    console.log('connected')
})
.catch(error => { 
    console.error(error)})


module.exports = mongoose;
```



Now we can call that file in order to establish the connections:

- in `app.js`: 

```javascript 

const mongoose = require ('./database');
```



### Environment variables

You have two different environments: dev and production. In order to be able to mantain the connexions in both parts of the application (for example, if we deply to Heroku our app, where the routes will change) we need to establish our defined environment for development in our computers. To do that, we will create environment variables.

+ In terminal inside: `project-folder/server`:

  `$ npm i dotenv`



+ in ``project-folder/server` - `.env: ` 

  ```MONGODB_URI = mongodb://localhost:27017/database-name```

  

### Install `bcrypt`

Let's install bcrypt to keep our user's passwords secure!

+ in the terminal, route  `project-folder/server:`

  `$ npm i bcrypt `



### Uninstall `jade`

mmm... don't remember WTF problem Thor had during the exercise, but you need to uninstall Jade or the world will explode: 

+ in the terminal, route  `project-folder/server:`

  `$ npm uninstall jade`



### Aditional

You can delete your `project-folder/server/views` folder, as we won't need any rendering service from the backend!.





## Auth

We are about to code in our Express.js app so we can create the **Signup/Login/Logout** functionalities. 



### Configurate the routes 

Let's configurate the routes that our API will use to retrieve information from the database:

+ in `app.js`

```typescript

const authRouter = require('./routes/auth');

...

app.use('/api/auth', authRouter);
```



+ in `auth.js`:

```typescript

'use strict';

const express = require('express');
const bcrypt = require('bcrypt');
const router = express.Router();

const User = require('../models/user');

router.get('/me', (req, res, next) => {
  if (req.session.currentUser) {
    res.json(req.session.currentUser);
  } else {
    res.status(404).json({code: 'not-found'});
  }
});

router.post('/login', (req, res, next) => {
  if (req.session.currentUser) {
    return res.status(401).json({code: 'unauthorized'});
  }

  const username = req.body.username;
  const password = req.body.password;

  if (!username || !password) {
    return res.status(422).json({code: 'validation'});
  }

  User.findOne({ username })
    .then((user) => {
      if (!user) {
        return res.status(404).json({code: 'not-found'});
      }
      if (bcrypt.compareSync(password, user.password)) {
        req.session.currentUser = user;
        return res.json(user);
      } else {
        return res.status(404).json({code: 'not-found'});
      }
    })
    .catch(next);
});

router.post('/signup', (req, res, next) => {
  if (req.session.currentUser) {
    return res.status(401).json({code: 'unauthorized'});
  }

  const username = req.body.username;
  const password = req.body.password;

  if (!username || !password) {
    return res.status(422).json({code: 'validation'});
  }

  User.findOne({username}, 'username')
    .then((userExists) => {
      if (userExists) {
        return res.status(422).json({code: 'username-not-unique'});
      }

      const salt = bcrypt.genSaltSync(10);
      const hashPass = bcrypt.hashSync(password, salt);

      const newUser = User({
        username,
        password: hashPass
      });

      return newUser.save()
        .then(() => {
          req.session.currentUser = newUser;
          res.json(newUser);
        });
    })
    .catch(next);
});

router.post('/logout', (req, res) => {
  req.session.currentUser = null;
  return res.status(204).send();
});

module.exports = router;
```





### Create the model

We need to create the model we'll be using in our database:

+ in `models/user.js`:

```javascript

'use strict';

const mongoose = require("mongoose");
const Schema   = mongoose.Schema;

const userSchema = new Schema({
  username: {
    type: String,
    required: true
  },
  password: {
    type: String,
    required: true
  }
}, {
  timestamps: true
});

const User = mongoose.model('User', userSchema);
```





### Configurate the user session

#### Add `express-session`

Let's install express-session (session middleware for Express) so we can keep track of the user's sessions: 

- in the terminal, route  `project-folder/server:`

  `$ npm install --save express-session connect-mongo`

  

+ in `server/app.js`

 ```javascript

const session = require('express-session');
const MongoStore = require('connect-mongo')(session);

...

app.use(session({
  store: new MongoStore({
    mongooseConnection: mongoose.connection,
    ttl: 24 * 60 * 60 // 1 day
  }),
  secret: 'authlivecode',
  resave: true,
  saveUninitialized: true,
  cookie: {
    maxAge: 24 * 60 * 60 * 1000
  }
}));
 ```



**NOTE**: The session is established between the Angular Server (localhost 4200) and the browser, so, at the moment we cannot connect the session with teh Server API.  Behind the scenes, the browser in establishing an AJAX connection with the Angular Server. 

We can solve this problem by using Tokens. Is an alphanumeric code with a encypted part and a codified part. You ask for the token to the server api and when reciving it, and you store it in the local storage. 

How is the authentication? We must send the token, check if is right and in that case keep working.

**AJAX -> Token -> Server API -> check if it's right**



## Front-End

Let's create the Login, Signup and Logout function for our app, connected to what we did in our Backend server.



### Routing  

We need to define the routes that will link the routes we created in our API to our Frontend: 

+ in `app.module.ts`

```typescript

import { RouterModule, Routes } from '@angular/router';

...

const routes: Routes = [
{ path: '',  component: HomePageComponent },
{ path: 'login',  component: LoginPageComponent },
{ path: 'signup',  component: SignupPageComponent },
 { path: 'private',  component: PrivatePageComponent },
];

...

imports: [ ..., RouterModule.forRoot(routes), ...]
```



### Generate Components 

We are going to create the components: home-page + login-page to be displayed when accesing the created routes in the browser: 

+ in the terminal, route  `project-folder/client:`

  `$ ng g c pages/login-page`
  `$ ng g c pages/home-page`
  `$ ng g c pages/signup-page`

  

  Remember that when using the `RouterModule` you need to set the `router-outlet`:

+ in `app.component.html`

  ````````````
  
  <router-outlet></router-outlet>
  ````````````



### Generate the service

We are going to create a service so we can storage our 'connect-with-API' functions in just one place: 

+ in the terminal, route  `project-folder/client:`

  `$ ng g s services/auth`



+ in `app.modules`:

```javascript

import { HttpClientModule } from '@angular/common/http';
import { AuthService } from './services/auth.service';

...

 imports: [
    HttpClientModule, 
  ]

...

  providers: [AuthService],

```



+ in `project-folder/server` in `auth.js` 

```javascript

router.get('/me', (req, res, next) => {
  if (req.session.currentUser) {
    res.json(req.session.currentUser);
  } else {
    res.status(404).json({code: 'not-found'});
  }
});
```



#### Configure the environment: 

+ in `environment.ts`:

```typescript

export const environment = {
  production: false,
  API_URL: 'http://localhost:3000/api/auth';
};
```



+ In `auth.services.js`:

```javascript

import { Injectable } from '@angular/core';
import { HttpClient, HttpResponse } from '@angular/common/http';
import { Subject, Observable } from 'rxjs';

import { environment } from '../../environments/environment';


@Injectable()
export class AuthService {

  private user: any;
  private userChange: Subject<any> = new Subject();

  private API_URL = environment.API_URL;

  userChange$: Observable<any> = this.userChange.asObservable();

  constructor(private httpClient: HttpClient) { }

  private setUser(user?: any) {
    this.user = user;
    this.userChange.next(user);
    return user;
  }

  me(): Promise<any> {
    const options = {
      withCredentials: true
    };
    return this.httpClient.get(`${this.API_URL}/me`, options)
      .toPromise()
      .then((user) => this.setUser(user))
      .catch((err) => {
        if (err.status === 404) {
          this.setUser();
        }
      });
  }

  login(user: any): Promise<any> {
    const options = {
      withCredentials: true
    };
    return this.httpClient.post(`${this.API_URL}/login`, user, options)
      .toPromise()
      .then((data) => this.setUser(data));
  }

  signup(user: any): Promise<any> {
    const options = {
      withCredentials: true
    };
    return this.httpClient.post(`${this.API_URL}/signup`, user, options)
      .toPromise()
      .then((data) => this.setUser(data));
  }

  logout(): Promise<any> {
    const options = {
      withCredentials: true
    };
    return this.httpClient.post(`${this.API_URL}/logout`, null, options)
      .toPromise()
      .then(() => this.setUser());
  }

  getUser(): any {
    return this.user;
  }
}
```





#### Observables

A observable is an object that can emmit other objects. 

```javascript

private userChange: Subject<any> = new Subject(); - OBSERVER
userChange$: Observable<any> = this.userChange.asObservable(); - OBSERVABLE
```

Subject is an object that acts like an Observer/Observable at the same time. 



+ in `app.component.html`:

```html

<!-- <header id="site-header">
  <div class="container"> -->

    <div [ngClass]="{'is-user': user, 'is-anon': anon}">
      <div *ngIf="loading">
        <span class="username text">loading...</span>
      </div>
      <div *ngIf="!loading && !!user">
        <a [routerLink]="['/profile']">{{user.username}}</a>
        <a (click)="logout()">logout</a>
      </div>
      <div *ngIf="!loading && anon">
        <a [routerLink]="['/login']">login</a>
        <a [routerLink]="['/signup']">signup</a>
      </div>
    </div>

  <!-- </div>
</div> -->
```



+ in `app.component.ts`:

```javascript

import { Router } from '@angular/router';
import { Component, OnInit } from '@angular/core';

import { AuthService } from './services/auth.service';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent implements OnInit {
  title = 'app';
  loading = true;
  anon: boolean;
  user: any;

  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  ngOnInit() {
    this.authService.userChange$.subscribe((user) => {
      this.loading = false;
      this.user = user;
      this.anon = !user;
    });
  }

  logout() {
    this.authService.logout()
      .then(() => this.router.navigate(['/login']));
  }
}
```





### Guards

Guards determine if the user is allowed or forbidden to access a certain route.



Let's generate the guards: 

- in the terminal, route  `project-folder/client`

  `$ ng g g guards/init-auth`

  `$ ng g g guards/require-anon`

  `$ ng g g guards/require-user`

  

+ register the guards in `app.module.ts`

```typescript

const routes: Routes = [
{ path: '',  component: HomePageComponent, canActivate: [ InitAuthGuard ] },
{ path: 'login',  component: AuthLoginPageComponent, canActivate: [ RequireAnonGuard ] },
{ path: 'signup',  component: AuthSignupPageComponent, canActivate: [ RequireAnonGuard ] },
{ path: 'page',  component: ... , canActivate: [ RequireUserGuard ] },
{ path: '**' , redirect to: }
]

...
    
providers: [...
  ...,
  InitAuthGuard,
  RequireAnonGuard,
  RequireUserGuard,
  ...
],
```



+ in `init-auth.guard.ts`:

```typescript

import { Injectable } from '@angular/core';
import { CanActivate } from '@angular/router';

import { AuthService } from '../services/auth.service';

@Injectable()
export class InitAuthGuard implements CanActivate {

  constructor(private authService: AuthService) { }

  canActivate(): Promise<any> {
    return this.authService.me()
      .then((user) => {
        return true;
      })
      .catch((error) => {
        console.error(error);
        return false;
      });
  }
}
```



+ in `require-user.guards.ts`:

```typescript

import { Injectable } from '@angular/core';
import { CanActivate } from '@angular/router';
import { Router } from '@angular/router';

import { AuthService } from '../services/auth.service';

@Injectable()
export class RequireUserGuard implements CanActivate {

  constructor(
    private authService: AuthService,
    private router: Router
  ) { }

  canActivate(): Promise<any> {
    return this.authService.me()
      .then((user) => {
        if (user) {
          return true;
        } else {
          this.router.navigate(['/login']);
          return false;
        }
      })
      .catch((error) => {
        console.error(error);
        return false;
      });
  }
}
```



+ in `require-anon.guard.ts`:

```typescript

import { Injectable } from '@angular/core';
import { CanActivate } from '@angular/router';
import { Router } from '@angular/router';

import { AuthService } from '../services/auth.service';

@Injectable()
export class RequireAnonGuard implements CanActivate {

  constructor(
    private authService: AuthService,
    private router: Router
  ) { }

  canActivate(): Promise<any> {
    return this.authService.me()
      .then((user) => {
        if (!user) {
          return true;
        } else {
          this.router.navigate(['/']);
          return false;
        }
      })
      .catch((error) => {
        console.error(error);
        return false;
      });
  }
}
```





### Cors

- in the terminal, route  `project-folder/server`:

  `$ npm install --save cors`

  

- in `app.js`:

```typescript

const cors = require('cors');

...

app.use(cors({
  credentials: true,
  origin: ['http://localhost:4200']
}));
```

Note: If you are using dotenv module with environment variables you could change origin to `process.env.CORS_URL`



### Login form

We need to create a form so our users can log into the app!



Let's generate the component: 

- in the terminal, route  `project-folder/client`

  `$ ng g c components/login`



+ in `app.module.ts`:

```typescript

import { FormsModule } from '@angular/forms';

...

@NgModule({
   ...
  imports: [
    ...,
    FormsModule,
    ...
  ],
  ...
})
```



+ in logincomponent.ts 

```typescript

import { Component, OnInit } from '@angular/core';
import { Router } from '@angular/router';


@Component({
  selector: 'app-my-form',
  templateUrl: './my-form.component.html',
  styleUrls: ['./my-form.component.css']
})
export class MyFormComponent implements OnInit {

username: string;
password: string;

  constructor( 
    private authService: AuthService,
    private router : Router
    ) { }

  ngOnInit() {
  }

  submitForm(form) {
    this.authService.login({
        username: this.username,
        password: this.password
    })
    .then(()=> {
        this.router.navigate(['/private'])
    })
    .catch(error => {
        console.log(error)
    })  
  }

}
```



+ in logincomponent.html

```typescript

<form (ngSubmit)="submitForm(form)" #form="ngForm">

  <div class="field" [ngClass]="{'has-error': feedbackEnabled && usernameField.errors}">
    <label>username</label>
    <input type="text" name="username" [(ngModel)]="username" #usernameField="ngModel" required />

    <label>password</label>
    <input type="password" name="password" [(ngModel)]="password" #passwordField="ngModel" required />

  </div>

  <div class="field submit">
    <button type="submit">Login</button>
  </div>
</form>
```





