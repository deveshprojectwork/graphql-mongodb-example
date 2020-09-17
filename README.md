# GraphQL-MongoDB-Example

All the important code is in `src/start.js`.

Install, build and run:

```
yarn install
yarn run build
yarn start
```

For Local Development 

You need to start Mongodb for Local development 

```
npm run startdev
```
https://medium.com/the-ideal-system/graphql-and-mongodb-a-quick-example-34643e637e49

GraphQL and MongoDB — a quick example
Nick Redmark
Nick Redmark
Follow
Dec 12, 2016 · 4 min read



A quick example with Apollo server for Express, mutations, MongoDB resolvers including support for nested queries in about 100 lines of code.
GraphQL documentation out there tends to focus on queries, less on mutations, less on defining a schema, even less on setting up a server, and even less less on binding the server to an actual database.
The problem is: to actually experience the beauty and the power of this tool, you need to go through the whole process from the bottom up. You need to see the full stack.
This post goes from defining a schema, wiring a MongoDB with a GraphQL server, to finally querying it via the GraphiQL-GUI.
Don’t read this post
My advice: don’t read this post. Instead, take 5 minutes (0.5 minutes if you use yarn :P) and clone the project, build and run it, and play with the graphiql interface yourself on http://localhost:3001/graphiql.
Then check out the about 100 lines of source code.
Only then come back here and have it all explained.
The schema
Let’s start with a graphql schema
const typeDefs = [`
  // The following sections go in here
`]
You have Posts, which have a title and content, and Comments, which just have a content. Each comment belongs to a post.
type Post {
  _id: String
  title: String
  content: String
  comments: [Comment]
}
type Comment {
  _id: String
  postId: String
  content: String
  post: Post
}
You want to be able to fetch single posts and comments by id, as well as a list of all posts.
type Query {
  post(_id: String): Post
  posts: [Post]
  comment(_id: String): Comment
}
You want to be able to create posts and comments.
type Mutation {
  createPost(title: String, content: String): Post
  createComment(postId: String, content: String): Comment
}
This completes the graphql schema
schema {
  query: Query
  mutation: Mutation
}
MongoDB
Of course you’ll need a running mongodb instance.
We create a couple of MongoDB collections
import {MongoClient, ObjectId} from 'mongodb'
const MONGO_URL = 'mongodb://localhost:27017/blog'
const db = await MongoClient.connect(MONGO_URL)
const Posts = db.collection('posts')
const Comments = db.collection('comments')
(in case you wonder, see how I configured await in the .babelrc file. Remember await statements need to be inside an async function)
Now we need to map mongodb with graphql.
const resolvers {
  // All resolvers go in here
}
We start with the queries:
Query: {
  post: async (root, {_id}) => {
    return prepare(await Posts.findOne(ObjectId(_id)))
  },
  posts: async () => {
    return (await Posts.find({}).toArray()).map(prepare)
  },
  comment: async (root, {_id}) => {
    return prepare(await Comments.findOne(ObjectId(_id)))
  },
},
If you wonder, prepare is just a little function that transforms _id from an ObjectId into a string.
const prepare = (o) => {
  o._id = o._id.toString()
  return o
}
But how to do joins? How to get a post’s comments? Or a comment’s post? With these resolvers:
Post: {
  comments: async ({_id}) => {
    return (await Comments.find({postId: _id}).toArray()).map(prepare)
  }
},
Comment: {
  post: async ({postId}) => {
    return prepare(await Posts.findOne(ObjectId(postId)))
  }
},
How do you create posts and comments? Finally here are the mutations:
Mutation: {
  createPost: async (root, args, context, info) => {
    const res = await Posts.insert(args)
    return prepare(await Posts.findOne({_id: res.insertedIds[1]}))
  },
  createComment: async (root, args) => {
    const res = await Comments.insert(args)
    return prepare(await Comments.findOne({_id: res.insertedIds[1]}))
  },
},
Apollo GraphQL server for express
We put the schema and the resolvers together like
import {makeExecutableSchema} from 'graphql-tools'
...
const schema = makeExecutableSchema({
  typeDefs,
  resolvers
})
and we bind graphql and graphiql to an express server like this:
import {graphqlExpress, graphiqlExpress} from 'graphql-server-express'
import {makeExecutableSchema} from 'graphql-tools'
const PORT = 3001
...
const app = express()
app.use('/graphql', bodyParser.json(), graphqlExpress({schema}))
app.use('/graphiql', graphiqlExpress({
  endpointURL: '/graphql'
}))
app.listen(PORT, () => {
  console.log(`Visit http://localhost:${PORT}`)
})
That’s it. Again, to get the setup right, just clone this repo, build and run it.
Fun with GraphQL queries
Finally we get to appreciate the beauty of GraphQL.
Visit http://localhost:3001/graphiql and play around with it.
For example, add a post:
mutation {
  createPost(title:"hello", content:"world") {
    _id
    title
    content
  }
}
then run:
query {
  posts {
    _id
    title
    content
  }
}
or
query {
  post(_id:"584ebf8bee8d98127efb080c") {
    _id
    title
    content
  }
}
But now on to the fancier stuff. What about nested queries and all that? Add a comment:
mutation {
  createComment(postId: "584ebf8bee8d98127efb080c", content: "I like the way you say hello world.") {
    _id
    postId
    content
  }
}
Now try getting all posts again, this time including comments
query {
  posts {
    _id
    title
    content
    comments {
      _id
      postId
      content
    }
  }
}
And you should see something like
{
  "data": {
    "posts": [
      {
        "_id": "584ebf8bee8d98127efb080c",
        "title": "hello",
        "content": "world",
        "comments": [
          {
            "_id": "584ec10eee8d98127efb080e",
            "postId": "584ebf8bee8d98127efb080c",
            "content": "I like the way you say hello world."
          }
        ]
      },
    ]
  }
}
And you can go on, by including the post in each comment…
query {
  posts {
    _id
    title
    content
    comments {
      _id
      postId
      content
      post {
        _id
        title
        content
      }
    }
  }
}
ok this is getting redundant, but you get the point
{
  "data": {
    "posts": [
      {
        "_id": "584ebf8bee8d98127efb080c",
        "title": "hello",
        "content": "world",
        "comments": [
          {
            "_id": "584ec10eee8d98127efb080e",
            "postId": "584ebf8bee8d98127efb080c",
            "content": "I like the way you say hello world.",
            "post": {
              "_id": "584ebf8bee8d98127efb080c",
              "title": "hello",
              "content": "world"
            }
          }
        ]
      },
    ]
  }
}
Try it out, it’s quick and fun
Conclusion: graphql is very powerful, but it’s still somewhat tricky to get it all set up. I hope this quick example was helpful to you.
Again, here is the repo. If it was useful encourage others to try it in the comments. If you know a better way to wire a mongodb to graphql please let me know. We are all in the same boat.
I’m in the process of evaluating new technologies to create my favorite web-development stack. Currently trying to replace meteor. For example, I wrote about how next.js is promising as a universally-rendered frontend — but it needs a separate backend for the data, which is why I’m taking a look at graphql. Follow me to hear updates.
Update: I created a node.js (next.js) starter library that features a full accounts system, called Staart. Check it out live here!
