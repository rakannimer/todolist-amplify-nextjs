
---

title: SSR Web App with NextJS & AWS Amplify

published: false

description: From zero to an infinitely scaleable web app in 5 minutes

tags: AWS, Amplify, NextJS

---

NextJS makes server-side rendering easy wherever your data is coming from.

AWS Amplify is a library and toolchain that makes it a breeze to setup, manage and use infinitely scaleable cloud infrastructure from AWS.

You don't need to be familiar with the rest of AWS services to use it, however if you are, you'll notice that Amplify offers a layer of abstraction over popular and battle tested AWS cloud services like AppSync, DynamoDB, Cognito, Lambda and many others. Amplify packages these cloud services under categories like Analytics, Auth, API, Storage, PubSub... If you'd like to know more about it, make sure to check out [their site](https://aws-amplify.github.io/).

Please note that you can deploy a production ready app without ever needing to know or manually manage any of these services. AWS Amplify can be your only contact point with the cloud.

In this post we will build a server-side rendered todo app that many users can edit at the same time. The app will consist of two pages :

1. A static page that shows the todos : /
2. A dynamic page that shows a single todo : /

## Creating our app skeleton

Let's start by setting up our directory and initializing git


```sh
mkdir todo-list
cd todo-list
npm init -y
git init

```

By now we have a directory that contains only our package.json with the defaults specified.

We can now install our dependencies
  
```sh

npm i react react-dom next immer nanoid
# If you're using typescript
npm i -D typescript -@types/react @types/react-dom @types/node
  
```

> Note that the [immer](https://github.com/immerjs/immer/) and [nanoid](https://github.com/ai/nanoid) dependencies are not necessary 

> but immer will make it easier for us to manipulate React state and 
 
> nanoid is a tiny util to generate a unique id for each to do.

And add 3 scripts to our `package.json`


```json
{
  "scripts":{ 
    "dev": "next",
    "build": "next build",
    "start": "next start"
  }
}
```

Next, we need to create a main page for the web application, 
when using Next.js we just need to create a directory called pages and put in it our main file as index.js (or index.tsx)

```sh
mkdir pages
touch pages/index.js # or pages/index.tsx
```

Our main page will just return the app shell to confirm our setup is correct.

```typescript
import * as React from "react";

const App = () => {
  return (
    <>
      <header>
        <h2>To Do List</h2>
      </header>
      <main>Hello World</main>
    </>
  );
};
export default App;
```

Let's run it now : 

```sh
npm run dev
```

Next.js will setup a tsconfig for us and start a server on localhost:3000

Visiting that should give us something like this : 

![Todo list skeleton displayed in the browser](https://thepracticaldev.s3.amazonaws.com/i/6tpcjz46o5y47r4v0m8k.png)


We're now ready to add the functionality for our app. 

It should have a text field with a button next to it and a list of edit-able and delete-able todos.

To manage the state we will use `React.useReducer` with initial state equal to : 
```js
{
  currentTodo:"",
  todos: [] 
}
```
and the reducer will support 4 actions `add`, `update`, `set-current` and `delete`

Our reducer :

```typescript
import produce from "immer";

/*<IfTypescript>*/
type Todo = {
  id: string;
  name: string;
  createdAt: string;
  completed: boolean;
};
type State = { todos: Todo[]; currentTodo: string };
type Action =
  | { type: "add" | "update" | "delete"; payload: Todo }
  | { type: "set-current"; payload: string };
/*</IfTypescript>*/

const reducer/*: React.Reducer<State, Action>*/ = (state, action) => {
  switch (action.type) {
    case "add": {
      return produce(state, draft => {
        draft.todos.push(action.payload);
      });
    }
    case "update": {
      const todoIndex = state.todos.findIndex(
        todo => todo.id === action.payload.id
      );
      if (todoIndex === -1) return state;
      return produce(state, draft => {
        draft.todos[todoIndex] = { ...action.payload };
      });
    }
    case "delete": {
      const todoIndex = state.todos.findIndex(
        todo => todo.id === action.payload.id
      );
      if (todoIndex === -1) return state;
      return produce(state, draft => {
        draft.todos.splice(todoIndex, 1);
      });
    }
    default: {
      throw new Error(`Unsupported action ${JSON.stringify(action)}`);
    }
  }
};
```

And the UI component : 

```typescript

const App = () => {
  // The reducer defined before
  const [state, dispatch] = React.useReducer(reducer, {
    currentTodo: "",
    todos: []
  });
  return (
    <>
      <header>
        <h2>To Do List</h2>
      </header>
      <main>
        <form
          onSubmit={event => {
            event.preventDefault();
            dispatch({
              type: "add",
              payload: {
                id: nanoid(),
                name: state.currentTodo,
                completed: false,
                createdAt: `${Date.now()}`
              }
            });
            dispatch({ type: "set-current", payload: "" });
          }}
        >
          <input
            type="text"
            value={state.currentTodo}
            onChange={event => {
              dispatch({ type: "set-current", payload: event.target.value });
            }}
          />
          <button type="submit">Add</button>
        </form>
        <ul>
          {state.todos.map(todo => {
            return (
              <li key={todo.id}>
                <input
                  type={"text"}
                  value={todo.name}
                  onChange={event => {
                    dispatch({
                      type: "update",
                      payload: { ...todo, name: event.target.value }
                    });
                  }}
                />
                <button
                  onClick={() => {
                    dispatch({ type: "delete", payload: todo });
                  }}
                >
                  Delete
                </button>
              </li>
            );
          })}
        </ul>
      </main>
    </>
  );
};
```

At this point we have a working to do list app that works offline.
If you're following along with code, now might be a good time to create a commit before jumping into integrating our app with AWS Amplify.

> Before you commit make sure to add a .gitignore file  
> `printf "node_modules\n.next" > .gitignore`

