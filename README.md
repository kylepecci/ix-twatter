# ix-twatter


<p align="center">
  <img src="screens/3251932690_21d32c07e9_o.png" alt="Bundle Analyzer example">
</p>



## Getting Started

### Install Homebrew
Note: if you already have homebrew installed, you can skip this. 

Installing Homebrew is effortless, open Terminal and enter : (MAC)
```sh
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"

```
Note: Homebrew will download and install Command Line Tools for Xcode 8.0 as part of the installation process.

For windows users: https://dev.mysql.com/doc/workbench/en/wb-installing-windows.html

(let me know if you run into issues Max ;))


### Install MySQL

To install MySQL enter : `brew install mysql`

Install brew services first : `brew tap homebrew/services`

Load and start the MySQL service : `brew services start mysql`.
Expected output : Successfully started mysql (label: homebrew.mxcl.mysql)

(Max, good luck ma man.)

### Install Docker

Download Docker and install it.

You can find Docker for mac here: 
https://store.docker.com/editions/community/docker-ce-desktop-mac

And here it is for windows:
https://store.docker.com/editions/community/docker-ce-desktop-windows


### Get the server up and running

in a new terminal window, CD into your server, install dependencies and deploy prisma.

0. Globally install prisma and graphql-cli by running `npm insnstall -g prisma graphql-cli`.
1. `cd server/prisma`
2. run `yarn` to install depencies.
3. run `prisma deploy` to deploy prisma.
4. CD back into the server directory by running `cd ..` to go a directory up.

Now go ahead and run `yarn dev` and you should see your playground 🍾


### Get the front-end up and running

1. go into your front-end directory by doing `cd client`
2. install dependencies by running `yarn`
3. run the app by running `yarn start`

This should open a web app (react.js) running on port 3000 on localhost.

Note: We actually won't be using react.js much yet, expect this to be more relevant next week but it's good to make sure your front-end is running never the less!

---

## Prisma

Now that both our server and front-end are properly setup, let's start defining our schema. First, it's important to note that Prisma is generating Queries (reads) and Mutations (writes) for your database based on whatever you put in your datamodel  schema file inside `prisma/datamodel.graphql`. Prisma runs on it's own and is it's separate thing. You still have your own graphql server running besides that, and your graphql server is in between your Prisma server & your client (in this case our react.js application). 

This is important especially because Prisma literally generates all possible filters/sorting that you would want, on every type that you define and for every property. You will still need to pick and choose what you expose and perhapse define some custom logic in your own graphql-server if your needs are more than a simple read/write. 

In other words, your Prisma service itself is never actually exposed to the client, only your graphql service can talk to it. 

Here is the diagram from Prisma's own website to reiterate this point:


![alt text](screens/prisma-hero-snip1.png "Logo Title Text 1")

As you can see, Prisma is sitting in front of your database(s).

---

## Designing your *database* schema

We're building a stripped down and simplified version of twitter. To be clear, the only features we are going to care about are the following:

- Authentication.
- Posting tweets.
- Following.
- Uploading a profile picture. (BONUS FEATURE! 🥓)

First, let's start by defining a User type. Our app has users, just like twitter's 😎.

```gql
type User {
  id: ID! @unique
  email: String! @unique
  password: String!
  name: String!
  picture: String
  tweets: [Tweet!]!
}
```

Most of these properties should be self explanatory, our user will have an ID and it's unique. By adding the `@unique` directive to our property, we let Prisma know that there can not be more than 1 user with the same ID. We do the same thing for emails. Also, our user has a bunch of tweets, so we attach a `tweets` property that will have an array of tweets, as you can see here : `tweets: [Tweet!]!`

Finally, our User also has an email and password because they will need to `authenticate` and log in. We don't want no anonymous tweets! This isn't blockchain.

Hold on! We almost forgot to create the Tweet type. Let's make sure this is defined:

```gql
type Tweet {
  id: ID! @unique
  createdAt: DateTime!
  updatedAt: DateTime!
  text: String!
  author: User!
}
```

In this case, the main point is that a Tweet is owned by a User and we define that relationship by giving the Tweet type an `author` property that links to the `User` type.

Now, in order to tell Prisma to make these changes run `prisma deploy` in your terminal. (make sure you are in the server repo).

Here is the expected output you should see in your terminal: 

![alt text](screens/prisma-changes-1.png "Logo Title Text 1")

Finally, run `yarn build` as well in order to complile your typescript.


That's it for our schema for now. We'll get back to this and add some more things as we go! Now let's start creating some tweets and users.

## Authentication

The authentication part of the code is done for you, but let's walk through it step by step so that you understand it. 

You will have to eventually allow your users to log in on the front-end!


So the first thing we are going to look at is our schema.graphql file. Let's open it up and see what's inside there so far:

```gql
# import * from "./generated/prisma.graphql"

type Query {
  feed: [Tweet!]!
  tweet(id: ID!): Tweet
  me: User
}

type Mutation {
  signup(email: String!, password: String!, name: String!): AuthPayload!
  login(email: String!, password: String!): AuthPayload!
}

type AuthPayload {
  token: String!
  user: User!
}

type User {
  id: ID!
  email: String!
  name: String!
  tweets: [Tweet!]!
}
```

As you can see, we have two mutations here. `signup` and `login`. They both take an email and a password and return an `AuthPayload` we've defined. This is what tells graphql what our mutations will look like, what parameters they take etc. We will now look at their *implementations*.

To see that, go to `src/Mutation/auth.ts`. Here's what it looks like:

```ts
import * as bcrypt from 'bcryptjs'
import * as jwt from 'jsonwebtoken'
import { Context } from '../../utils'

export const auth = {
  async signup(parent, args, ctx: Context, info) {
    const password = await bcrypt.hash(args.password, 10)
    const user = await ctx.db.mutation.createUser({
      data: { ...args, password },
    })

    return {
      token: jwt.sign({ userId: user.id }, process.env.APP_SECRET),
      user,
    }
  },

  async login(parent, { email, password }, ctx: Context, info) {
    const user = await ctx.db.query.user({ where: { email } })
    if (!user) {
      throw new Error(`No such user found for email: ${email}`)
    }

    const valid = await bcrypt.compare(password, user.password)
    if (!valid) {
      throw new Error('Invalid password')
    }

    return {
      token: jwt.sign({ userId: user.id }, process.env.APP_SECRET),
      user,
    }
  },
}
```

Let's break this signup function down line by line:

```ts
  async signup(parent, args, ctx: Context, info) {
    const password = await bcrypt.hash(args.password, 10)
    const user = await ctx.db.mutation.createUser({
      data: { ...args, password },
    })

    return {
      token: jwt.sign({ userId: user.id }, process.env.APP_SECRET),
      user,
    }
  },
```

First, we see our common arguments. Parent, args & ctx. These should already be familiar to you. As you can see, we pass `password` to this `bcrypt.hash` function which will take our user's password and hash it to make it secure. We've covered this a bit during our slides when we spoke about Authentication. All of this is the following: 

```ts
const password = await bcrypt.hash(args.password, 10)
```

With the encrypted password, we then pass that to our `createUser` mutation which was created for us by Prisma. We are able to access all our mutations via this `db.mutation` object provided to us inside `ctx`. This will be very useful througout the project and you will use it a lot. If you ever need to access queries or mutations outside of your playground/client without using graphql syntax, ctx.db is your friend!

Here's the line of code:

```ts
const user = await ctx.db.mutation.createUser({
  data: { ...args, password },
})
```

We've now created the user with the hashed password. If you are seeing `...args` and are unfamiliar with it, this is called the spread operator in Javascript. It allows us to "spread" an object instead of having to insert assign every property of an object to a new object one by one.

All we are doing is simply passing the arguments, `email & password` and spreading it for a nicer/cleaner syntax. To be clear, the above is the equivalent of this:

```ts
const user = await ctx.db.mutation.createUser({
  data: { email: args.email, password: args.password },
})
```

Makes sense?
Please watch this video on the topic before you continue:
https://www.youtube.com/watch?v=Y7pVoDkwVLY

Now that we've finally created the user and hashed their password, we should give the user back a token (our secret that they can share with our server in order to be authorized). 

This is what that looks like:

```ts
  return {
      token: jwt.sign({ userId: user.id }, process.env.APP_SECRET),
      user,
    }
```

The way the token is generated is using this `jwt.sign` function. What this does is, using our `APP_SECRET` (This IS a very secret you should never share with the outside world. Think of it like a master password), the library `jwt` has generated a signed token using the user's `id` in our database (challenge question: why not use email instead?) and our secret. 

Great, so now our `Mutation` object that is imported in `src/index.ts` has a signup method and our `schema.graphql` has also modeled it. This means that we can start using it!

Go ahead and create a user in your server. 

With the following query: 
```gql
mutation {
  signup(
    name: "Harris Robin"
    email: "harris@me.com"
    password: "password123"
  ) {
    token
    user {
      email
      name
      id
    }
  }
}
```
You should see something like this:

![alt text](screens/create-user-mutation.png "Logo Title Text 1")

A user was created, we have their information and a *token*. 
More on this *token* business later though! 



<p align="center">
  <img src="https://media.giphy.com/media/xT0xeJpnrWC4XWblEk/giphy.gif" alt="Bundle Analyzer example" width="650" height="335">
</p>


Now your turn!


1. Start by creating one of your tweet mutations! This one will be `createTweet`. It should do exactly what it sounds like and create a tweet.
  1. The way you create this mutation in graphql should be partly familiar to you and remain the same. What's different is that you now have a database, so remember to use that in your resolver! 
  2. Here's how creating a tweet would look like in your resolvers:
    ```ts
      const user = await ctx.db.mutation.createTweet({
        data: { 
          text: args.text,
          author: {
            connect: {
              email: "harris@me.com" // make sure you use your created user's email!
            }
          }
        },
      })
    ```

2. Make a feed query that returns a list of all the tweets (this is our twitter feed!)

3. Make a tweet query that returns a single tweet using an ID!
