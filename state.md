# Managing both Server and Local Ngrx State
### Directory Strucuture
```
src/
   app/
      components/
      containers/
         user.ts
      effects/
         user.ts
      store/
         actions/
            server/
               user.ts
            local/
               user.ts
         models/
            user.ts
         reducers/
            user.ts
```
#### Example:
For this example we will be creating the basic parts of a `User` service.

## A service consists of:
- `Container`
    - Houses the Components
- `Models`
    - Represents a Row from a Table
- `Actions`
    - Does something to the State
- `Effects`
    - Makes external API Requests
- `Reducer`
    - Changes the local in browser state

### The `Container`
The `Container` is the entry point from a url request. For our User example our path will be `/users`, when a user navigates to that path, angular will load the associated container. (Don't forget to build out the route in the routing module).
#### Creating the `Container`
`ng g component containers/{service name}`
ie.
`ng g component containers/user`

### The `Models`
Models are Objects that represent data returned by the API. That data also represents the database structure so the properties of the object will represent the column names of the table.

`store/models/user.ts`
```typescript
export interface User {
   id: number;
}
```

### The `Actions`
An Action is literally that, an 'Action'. Something that is doing something. In this case it is updating state. That state
can either be on the Server (through API requests or `Effects` updating the database) or in the local memory of the browser (Redux Store).

```
store/actions/
    server/user.ts
    local/user.ts
```
`store/action/server/user.ts`
```typescript
import {Action} from "@ngrx/store";
import {User} from "../../models/user";

export const LOAD_ALL = '[User] Load All';
export const LOAD = '[User] Load';
export const ADD = '[User] Add';
export const SET = '[User] Set';
export const DELETE = '[User] Delete';

export class LoadAllUsersAction implements Action {
  readonly type = LOAD_ALL;
}

export class LoadUserAction implements Action {
  readonly type = LOAD;
  constructor(public payload: number) {} // user id
}

export class AddUserAction implements Action {
  readonly type = ADD;
  constructor(public payload: User) {}
}

export class SetUserAction implements Action {
  readonly type = SET;
  constructor(public payload: User) {}
}

export class DeleteUserAction implements Action {
  readonly type = DELETE;
  constructor(public payload: number) {} // user id
}

export type Actions =
  | LoadAllUsersAction
  | LoadUserAction
  | AddUserAction
  | SetUserAction
  | DeleteUserAction;
```

`store/action/local/user.ts`
```typescript
import {Action} from "@ngrx/store";
import {User} from "../../models/user";

export const ADD = '[User] Add';
export const SET = '[User] Set';
export const DELETE = '[User] Delete';

export class AddUserAction implements Action {
  readonly type = ADD;
  constructor(public payload: User) {}
}

export class SetUserAction implements Action {
  readonly type = SET;
  constructor(public payload: User) {}
}

export class DeleteUserAction implements Action {
  readonly type = DELETE;
  constructor(public payload: number) {} // user id
}

export type Actions =
  | AddUserAction
  | SetUserAction
  | DeleteUserAction;

```

Notice how we don't have any `Local Actions` actions. These are done through external calls to the server.

### `Effects`
Effects make external calls to API endpoints to update/retrieve data from the server then call associated `Local Actions` to update the local store.

#### Creating the `ApiService` Method
`services/api.service.ts`
```typescript
addUser(action: user.AddUserAction): Observable<number> {
    this.startLoading(action.type);
    return this.http.post(`${this.API_PATH}/users`, action.payload)
      .map(res => res.json().data.id)
      .catch(e => this.handleError(e))
      .finally(() => this.endLoading(action.type))
}
```
`effects/user.ts`
```typescript
import {Injectable} from "@angular/core";
import {Actions, Effect} from "@ngrx/effects";
import {Observable} from "rxjs/Observable";
import {Action} from "@ngrx/store";
import * as userServerActions from "./../store/actions/server/user";
import * as userLocalAction from "./../store/actions/local/user";
import * as errors from '../store/actions/errors';
import {ApiService} from "../services/api.service";
import {of} from "rxjs/observable/of";

@Injectable()
export class UserEffects {

  constructor(private actions$: Actions, private apiService: ApiService) {}

  @Effect()
  newUser$: Observable<Action> = this.actions$
    .ofType(userServerActions.ADD)
    .switchMap(a => {
      // cast the action so we don't get type script errors
      let action: userServerActions.AddUserAction = <userServerActions.AddUserAction>a;
      // listen for more events coming in
      const next$ = this.actions$.ofType(userServerActions.ADD).skip(1);
      // call the api service
      return this.apiService.addUser(action)
        .takeUntil(next$)
        .map(id => {
          // set the local store data
          return new userLocalAction.AddUserAction(Object.assign({}, action.payload, {id: id}))
        })
        .catch(e => of(new errors.ApiErrorAction(e)))
    })
}
```
### The `Reducer`
`store/reduces/user.ts`
```typescript
import {User} from "../models/user";
import * as user from "../actions/local/user";

export interface State {
  entities: {[id: number]: User},
  selectedUserId: number;
}

export const initialState: State = {
  entities: {},
  selectedUserId: 0,
};

export function reducer(state = initialState, action: user.Actions) {
  switch (action.type) {
    case user.ADD:
      let newUser = {};
      newUser[action.payload.id] = action.payload;
      return Object.assign({}, state, {entities: Object.assign({}, state.entities, newUser)})
  }
}

export const getUsers = (state: State) => state.entities;
```
Add the new reducer to the index reducer.
```typescript
import * as fromUser from './users';

export interface State {
  ...
  user: fromUser.State;
}

const reducers = {
  ...
  user: fromUser.reducer,
};

export const getUserState = (state: State) => state.user;
export const getUsers = createSelector(getUserState, fromUser.getUsers);
```

### Wrapping Up
Now we can finish things off by retrieving our loaded users from from the store, or dispatching a load request in our `user` container.

```typescript
import { Component, OnInit } from '@angular/core';
import {Store} from "@ngrx/store";
import * as fromRoot from "./../../store/reducers/index";
import * as serverUser from "./../../store/actions/server/user";
import {User} from "../../store/models/user";

@Component({
  selector: 'app-users',
  templateUrl: './users.component.html',
  styleUrls: ['./users.component.css']
})
export class UsersComponent implements OnInit {
  
  users: {[id:number]: User} = {};

  constructor(private store: Store<fromRoot.State>) { }

  ngOnInit() {
    this.store.select(fromRoot.getUsers).subscribe(users => {
      if (!users || !Object.keys(users).length) {
        // dispatch load
        this.store.dispatch(new serverUser.LoadAllUsersAction())
      } else {
        this.users = users
      }
    })
  }
}
```
