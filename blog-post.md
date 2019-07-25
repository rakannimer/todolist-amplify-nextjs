---
title: Server-Side Rendered Realtime Web App with Next.js & AWS Amplify

published: false

description: Going from an idea to an infinitely scale-able web app

tags: AWS, Amplify, Next.js, React, Javascript, Typescript

cover_image: https://thepracticaldev.s3.amazonaws.com/i/026cx817wn9ahvpnw4pb.png
---

In this blog post we will go through building a server-rendered realtime collaborative todo list app with Next.js and AWS Amplify.

You can check out the final code [here](https://github.com/rakannimer/todolist-amplify-nextjs)

- [Introduction](#introduction)
- [Creating our app skeleton](#creating-our-app-skeleton)
- [Adding offline functionality](#adding-offline-functionality)
- [Preparing the Graqhql Schema for Amplify GraphQL Transform](#preparing-the-graqhql-schema-for-amplify-graphql-transform)
- [Setting up AWS Amplify on your computer](#setting-up-aws-amplify-on-your-computer)
- [Creating the API](#creating-the-api)
- [Editing the backend](#editing-the-backend)
- [Saving Todos in the cloud](#saving-todos-in-the-cloud)
- [Fetching initial todos on the server-side](#fetching-initial-todos-on-the-server-side)
- [Listening to todos being added by others](#listening-to-todos-being-added-by-others)
- [Listening to todos modified and deleted by others](#listening-to-todos-modified-and-deleted-by-others)
- [Deploying our app](#deploying-our-app)

## Introduction

The app will have dynamic and static routes to demonstrate how to load and render data from the server based on the incoming request url. And it has subscriptions to changes on the data to show how to use AWS Amplify to seamlessly listen to remote data.

![Amplify and Next.js](https://thepracticaldev.s3.amazonaws.com/i/026cx817wn9ahvpnw4pb.png)

Next.js makes server-side rendering easy wherever your data is coming from.

AWS Amplify is a library and toolchain that makes it a breeze to setup, manage and use infinitely scale-able cloud infrastructure from AWS.

You don't need to be familiar with the rest of AWS services to use it, however if you are, you'll notice that Amplify offers a layer of abstraction over popular and battle tested AWS cloud services like AppSync, DynamoDB, Cognito, Lambda, S3 and many others. Amplify packages these cloud services under categories such as Analytics, Auth, API, Storage, PubSub... If you would like to know more about it, make sure to check out [their website](https://aws-amplify.github.io/).

Please note that you can deploy a production ready app without ever needing to know or manually manage any of these services. AWS Amplify can be your only contact point with the cloud.

With that said, let's get started !

## Creating our app skeleton

First, let's set up a directory and initialize it with git

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
  "scripts": {
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

Next.js will setup a tsconfig for us (if we're using Typescript) and start a server on localhost:3000

Visiting that should give us something like this :

![Todo list app skeleton displayed in the browser](https://thepracticaldev.s3.amazonaws.com/i/6tpcjz46o5y47r4v0m8k.png)

## Adding offline functionality

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

Looking at some code, our reducer :

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

const reducer /*: React.Reducer<State, Action>*/ = (state, action) => {
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
  const add = () => {
    dispatch({
      type: "add",
      payload: {
        id: nanoid(),
        name: currentTodo,
        completed: false,
        createdAt: `${Date.now()}`
      }
    });
    dispatch({ type: "set-current", payload: "" });
  };
  const edit = (todo /*:Todo*/) => {
    dispatch({ type: "update", payload: todo });
  };
  const del = (todo /*:Todo*/) => {
    dispatch({ type: "delete", payload: todo });
  };
  return (
    <>
      <header>
        <h2>To Do List</h2>
      </header>
      <main>
        <form
          onSubmit={event => {
            event.preventDefault();
            add(state.currentTodo);
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
                    edit({ ...todo, name: event.target.value });
                  }}
                />
                <button
                  onClick={() => {
                    del(todo);
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

![Working todo list screenshot](https://thepracticaldev.s3.amazonaws.com/i/x6cw0kj5jnaxmsq7oyrf.png)

Let's now sync our todos with the cloud to be able to share them and collaborate with others.

## Preparing the Graqhql Schema for Amplify GraphQL Transform

Let's very quickly go through what Amplify GraphQL Transform is.

> The GraphQL Transform provides a simple to use abstraction
> that helps you quickly create backends for your web and mobile applications on AWS.

With it we define our data model using the GraphQL SDL and the amplify cli takes care of :

1. Provisioning/Updating required infrastructure for CRUDL operations.
2. Generating code for client-side CRUDL-ing

Input : GraphQL Data Shape.
Output: Elastic Infrastructure and code to seamless-ly interact with it.

> CRUDL = Create Read Update Delete List

In our case the GraphQL schema is simple it consists of one Todo type and one TodoList type that contains a sorted list of todos :

```graphql
type Todo @model {
  # ! means non-null GraphQL fields are allowed to be null by default
  id: ID!
  name: String!
  createdAt: String!
  completed: Boolean!
  todoList: TodoList! @connection(name: "SortedList")
  userId: String!
}

type TodoList @model {
  id: ID!
  createdAt: String!
  # Array of Todos sorted by Todo.createdAt
  todos: [Todo] @connection(name: "SortedList", sortField: "createdAt")
}
```

> We store the schema as `schema.graphql` to be re-used later.

The `@model` directive in the GraphQL Transform schema tells Amplify to treat the to do as a model and store objects of that type in DynamoDB and automatically configure CRUDL queries and mutations using AppSync.

The `@connection` directive allows us to specify n-to-n relationships between our data types and sort it on the server-side.

Read more about GraphQL Transform and supported directives [here](https://aws-amplify.github.io/docs/cli-toolchain/graphql)

If you've already used Amplify you can skip directly to [Creating the API](#creating-the-api)

## Setting up AWS Amplify on your computer

1. [Sign up](https://portal.aws.amazon.com/billing/signup#/start) for an AWS account
2. Install the AWS Amplify cli:

```sh
npm install -g @aws-amplify/cli
```

3. Configure the Amplify cli

```sh
amplify configure
```

[Read More](https://aws-amplify.github.io/docs/cli-toolchain/quickstart?sdk=js)

## Creating the API

We start by initializing amplify in our project.

```sh
npm i aws-amplify
amplify init
#<Interactive>
? Enter a name for the project (todolist) todolist
? Enter a name for the environment dev # or prod
? Choose your default editor: <MY_FAVORITE_EDITOR>
? Choose the type of app that you\'re building javascript # even if you're using typescript
? What javascript framework are you using react
? Source Directory Path: src
? Distribution Directory Path: out # Next.js exports to the out directory
? Build Command:  npm run-script build
? Start Command: npm run-script start
? Do you want to use an AWS profile? (Y/n) Y # Or use default
? Please choose the profile you want to use default
Your project has been successfully initialized and connected to the cloud!
# 🚀 Ready
#</Interactive>
```

At this point 2 new folders should have been created : `src` and `amplify`
It's safe to ignore them for now.

Now that amplify is initialized we can add any of its services (Auth, API, Analytics ...)
For our use-case we just need to use the API module. So we add it to the project using :

```sh
amplify add api
? Please select from one of the below mentioned services GraphQL
? Provide API name: todolist
? Choose an authorization type for the API (Use arrow keys)
❯ API key
  Amazon Cognito User Pool
? Do you have an annotated GraphQL schema? (y/N) y # The one we saved earlier to schema.graphql
? Provide your schema file path: ./schema.graphql
```

The API configuration is ready we need to push to sync our cloud resources with the current configuration :

```
amplify push
? Are you sure you want to continue? (Y/n) Y
? Do you want to generate code for your newly created GraphQL API (Y/n) Y # This code incredibly speeds up development
? Choose the code generation language target
❯ javascript
  typescript
  flow
? Enter the file name pattern of graphql queries, mutations and subscriptions src/graphql/**/*.js
? Do you want to generate/update all possible GraphQL operations - queries, mutations and subscriptions (Y/n) Y
? Enter maximum statement depth [increase from default if your schema is deeply nested] 2
⠼ Updating resources in the cloud. This may take a few minutes...
# Logs explaining what's happening
✔ Generated GraphQL operations successfully and saved at src/graphql
✔ All resources are updated in the cloud

GraphQL endpoint: https://tjefk2x675ex7gocplim46iriq.appsync-api.us-east-1.amazonaws.com/graphql
GraphQL API KEY: da2-d7hytqrbj5cwfgbbnxavvm7xry
```

And that's it 🎉 ! Our whole backend is ready and we have the client-side code to query it.

# Editing the backend

1. Edit `amplify/backend/api/apiname/schema.graphql`.
2. Run `amplify push`
3. That's it 👍

# Saving Todos in the cloud

In pages/index We start by importing `API` and `graphqlOperation` from `aws-amplify`
and configure our amplify application with `src/aws-exports.js`

```typescript
import { API, graphqlOperation } from "aws-amplify";
import config from "../src/aws-exports";
API.configure(config);
// Should be a device id or a cognito user id but this will do
const MY_ID = nanoid();
```

Next, if you open `src/graphql/mutations` you'll see there's a createTodo string containing the GraphQL Mutation to create a new todo.

We import it and use it after dispatching the `add` action.

```typescript
const add = async () => {
  const todo = {
    id: nanoid(),
    name: state.currentTodo,
    completed: false,
    createdAt: `${Date.now()}`
  };
  dispatch({
    type: "add",
    payload: todo
  });
  // Optimistic update
  dispatch({ type: "set-current", payload: "" });
  try {
    await API.graphql(
      graphqlOperation(createTodo, {
        input: { ...todo, todoTodoListId: "global", userId: MY_ID }
      })
    );
  } catch (err) {
    // With revert on error
    dispatch({ type: "set-current", payload: todo.name });
  }
};
```

And that's it our todos are now being saved to a highly available DynamoDB instance billed by request.

## Fetching initial todos on the server-side

We want the list we're building and the data in it to be server-rendered and sent to the client.
So we can't use the React.useEffect hook to load the data and store it in state.

Using Next.js's `getInitialProps` async method we can fetch data from anywhere and pass it as props to our page component.

So adding one to our main page would look like this

```typescript
import { getTodoList, createTodoList } from "../src/graphql/queries";

// <TypescriptOnly>
import { GetTodoListQuery } from "../src/API";
// </TypescriptOnly>

App.getInitialProps = async () => {
  let result; /*: { data: GetTodoListQuery; errors: {}[] };*/
  try {
    // Fetch our list from the server
    result = await API.graphql(graphqlOperation(getTodoList, { id: "global" }));
  } catch (err) {
    console.warn(err);
    return { todos: [] };
  }
  if (result.errors) {
    console.warn("Failed to fetch todolist. ", result.errors);
    return { todos: [] };
  }
  if (result.data.getTodoList !== null) {
    return { todos: result.data.getTodoList.todos.items };
  }

  try {
    // And if it doesn't exist, create it
    await API.graphql(
      graphqlOperation(createTodoList, {
        input: {
          id: "global",
          createdAt: `${Date.now()}`
        }
      })
    );
  } catch (err) {
    console.warn(err);
  }
  return { todos: [] };
};
```

And in our App component we initialize our state with the props we sent in `getInitialProps`

```typescript
//<TypescriptOnly>
import { GetTodoListQuery } from '../src/API'
type Props = {
  todos: GetTodoListQuery["getTodoList"]["todos"]["items"];
}
//</TypescriptOnly>

const App = ({ todos }/*:Props */) => {
const [state, dispatch] = React.useReducer(reducer, {
  currentTodo: "",
  todos
});
```

If you try refreshing the page now, you should see that your todos are persisted between refreshs and they're sorted in the same order as they were before when they were added

## Listening to todos being added by others

After we render the app on the client we want to listen to data changes that originated from other users so we can update our UI accordingly.

We will be using GraphQL subscriptions to listen to when a todo is added, updated or deleted.

Fortunately this won't take more than a couple of lines to setup.

```typescript
import { onCreateTodo } from "../src/graphql/subscriptions";
/*
With TS we create an Observable type to describe the return type of a GraphQL subscription.
Hopefully in future releases of aws-amplify we will have generic types for API.graphql that will make this un-necessary.
*/
type Observable<Value = unknown, Error = {}> = {
  subscribe: (
    cb?: (v: Value) => void,
    errorCb?: (e: Error) => void,
    completeCallback?: () => void
  ) => void;
  unsubscribe: Function;
};

// In our function component
const App = props => {
  // bla
  React.useEffect(() => {
    const listener /*: Observable<{
      value: { data: OnCreateTodoSubscription };
    }> */ = API.graphql(graphqlOperation(onCreateTodo));
    const subscription = listener.subscribe(v => {
      if (v.value.data.onCreateTodo.userId === MY_ID) return;
      dispatch({ type: "add", payload: v.value.data.onCreateTodo });
    });
    return () => {
      subscription.unsubscribe();
    };
  }, []);
  // blabla
};
```

## Listening to todos modified and deleted by others

We'll start by subscribing to two new subscriptions `onUpdateTodo` and `onDeleteTodo`

```typescript
import {
  onCreateTodo,
  onUpdateTodo,
  onDeleteTodo
} from "../src/graphql/subscriptions";
// <ts>
import { OnUpdateTodoSubscription, OnDeleteTodoSubscription } from "../src/API";

type Listener<T> = Observable<{ value: { data: T } }>;
// </ts>
// In our function component
const App = props => {
  // bla
  React.useEffect(() => {
    const onCreateListener: Listener<OnCreateTodoSubscription> = API.graphql(
      graphqlOperation(onCreateTodo)
    );
    const onUpdateListener: Listener<OnUpdateTodoSubscription> = API.graphql(
      graphqlOperation(onUpdateTodo)
    );
    const onDeleteListener: Listener<OnDeleteTodoSubscription> = API.graphql(
      graphqlOperation(onDeleteTodo)
    );

    const onCreateSubscription = onCreateListener.subscribe(v => {
      if (v.value.data.onCreateTodo.userId === MY_ID) return;
      dispatch({ type: "add", payload: v.value.data.onCreateTodo });
    });
    const onUpdateSubscription = onUpdateListener.subscribe(v => {
      dispatch({ type: "update", payload: v.value.data.onUpdateTodo });
    });
    const onDeleteSubscription = onDeleteListener.subscribe(v => {
      dispatch({ type: "delete", payload: v.value.data.onDeleteTodo });
    });

    return () => {
      onCreateSubscription.unsubscribe();
      onUpdateSubscription.unsubscribe();
      onDeleteSubscription.unsubscribe();
    };
  }, []);
  // blabla
};
```

And here's what our end result, a collaborative real-time todo list looks like

![Two browser windows on the same url using the app and seeing changes of one reflected in the other](https://thepracticaldev.s3.amazonaws.com/i/lq0h5jxart3e3hcmdt5n.gif)

Our first page is done but we still need to have our individual todo page and link to it from our list.

We need our individual todos to be indexed by search engines so we will need to server-render the data in the todo from the id in the url.

To do that, we create a new Next.js dynamic route in `pages/todo/[id].(t|j)sx` and use the `getInitialProps` async method to populate it with data from our AWS Amplify datasource.

```typescript
import * as React from "react";
import { API, graphqlOperation } from "aws-amplify";

import { getTodo } from "../../src/graphql/queries";
import config from "../../src/aws-exports";
// <ts>
import { GetTodoQuery } from "../../src/API";
type Props = { todo: GetTodoQuery["getTodo"] };
// </ts>
API.configure(config);

const TodoPage = (props /*: Props*/) => {
  return (
    <div>
      <h2>Individual Todo {props.todo.id}</h2>
      <pre>{JSON.stringify(props.todo, null, 2)}</pre>
    </div>
  );
};

TodoPage.getInitialProps = async context => {
  const { id } = context.query;
  try {
    const todo = (await API.graphql({
      ...graphqlOperation(getTodo),
      variables: { id }
    })) as { data: GetTodoQuery; errors?: {}[] };
    if (todo.errors) {
      console.log("Failed to fetch todo. ", todo.errors);
      return { todo: {} };
    }
    return { todo: todo.data.getTodo };
  } catch (err) {
    console.warn(err);
    return { todo: {} };
  }
};

export default TodoPage;
```

And last, we add a link to every todo item

```jsx
<a href={`/todo/${todo.id}`}>Visit</a>
```

## Deploying our app

To deploy the app, we can either use [`export`](https://Next.js.org/learn/excel/static-html-export) from Next.js to export the app as a static website deploy-able anywhere or we can deploy it as a node server.

And that's it we have built an SEO friendly server-side rendered collaborative todo list using Next.js and AWS Amplify.

If you have any questions feel free to comment here or ping me on [twitter](https://twitter.com/rakannimer).

If you enjoyed reading this
