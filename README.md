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

Now, in order to tell Prisma to make these changes run `prisma deploy` in your terminal. (make sure you are in the server repo)