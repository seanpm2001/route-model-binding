# Route model binding
> Bind the route parameters with Lucid models and automatically query the database

Route model binding is a neat little way to remove one-liner Lucid queries from your codebase and use conventions to query the database during HTTP requests.

In the following example, we connect the route params `:post` and `:comments` with the arguments accepted by the `show` method.

- The value of the first param from the URL will be used to query the first typed hinted model on the `show` method, (ie Post).
- Similarly, the value of the second param will be used to query the second typed hinted model, (ie Comment).

> **Note**: The params and models are connected using the order in which they appear and not the name. Because, TypeScript decorators have no way to know the names of the arguments accepted by a method.

```ts
// Routes file
Route.get('posts/:post/comments/:comment', 'PostsController.show')
```

```ts
// Controller
import Post from 'App/Models/Post'
import Comment from 'App/Models/Comment'
import { bind } from '@adonisjs/route-model-binding'

export default class PostsController {
  @bind()
  public async show({}, post: Post, comment: Comment) {
    return { post, comment }
  }
}
```

## Setup
Install the package from the npm registry as follows.

```sh
npm i @adonisjs/route-model-binding

# yarn lovers
yarn add @adonisjs/route-model-binding
```

Next, configure the package by running the following ace command.

```sh
node ace configure @adonisjs/route-model-binding
```

The final step is to register the `RmbMiddleware` inside `start/kernel.ts` file.

```ts
Server.middleware.register([
  // ...other middleware
  () => import('@ioc:Adonis/Addons/RmbMiddleware'),
])
```

## Basic usage
Let's start with the most basic example and then tune up the complexity level to serve different use cases.

In the following example, we will bind the `Post` model with the first parameter in the `posts/:id` route.

```ts
Route.get('/posts/:id', 'PostsController.show')
```

```ts
import Post from 'App/Models/Post'
import { bind } from '@adonisjs/route-model-binding'

export default class PostsController {
  @bind()
  public async show({}, post: Post) {
    return { post }
  }
}
```

The params and models are matched in the order they are defined. So the first param in the URL matches the first type-hinted model in the controller method.

The match is not performed using the name of the controller method argument, because TypeScript decorators cannot read them (so the technical limitation leaves us with order based matching only).

## Changing the lookup key
By default, the primaryKey of the model is used to find a matching row in the database. You can either change that globally or change it for just one specific route.

### Change lookup key globally via model
After the following change, the post will be queried using the `slug` property and not the primary key. In a nutshell, the `Post.findByOrFail('slug', value)` query is executed.

```ts
class Post extends BaseModel {
  public static routeLookupKey = 'slug'
}
```

### Change lookup key for a single route.
In the following example, we define the lookup key directly on the route enclosed with parenthesis.

```ts
Route.get('/posts/:id(slug)', 'PostsController.show')
```

**Did you notice that our route now reads a bit funny?**\
The param is written as `:id(slug)`, which does not translate well. Therefore, with Route model binding we recommend using the model name as the route param, because we are not dealing with the `id` anymore, we are fetching model instance from the database.

Following is the better way to write the same route.

```ts
Route.get('/posts/:post(slug)', 'PostsController.show')
```

## Change lookup logic
You can change the lookup logic of a model by defining a static `findForRequest` method on the model itself. The method receives the following parameters.

- `ctx` - The HTTP context for the current request
- `param` - The parsed parameter. The parameter has following properties.
    - `param.name` - The normalized name of the parameter.
    - `param.param` - The original name of the parameter defined on the route.
    - `param.scoped` - If `true`, the parameter must be scoped to its parent model.
    - `param.lookupKey` - The lookup key defined on the route or the model.
    - `param.parent` - The name of the parent param.
- `value` - The value of the param during the current request.

In the following example, we query only published posts. Also, make sure that this method either returns an instance of the model or raise an exception.

```ts
class Post extends BaseModel {
  public static findForRequest(ctx, param, value) {
    const lookupKey = param.lookupKey === '$primaryKey' ? 'id' : param.lookupKey

    return this
      .query()
      .where(lookupKey, value)
      .whereNotNull('publishedAt')
      .firstOrFail()
  }
}
```

## Scoped params
When working with nested route resources, you might want to scope the second param as a relationship with the first param.

A great example of this is finding a post comment by id, but also making sure that it is a child of the post mentioned within the same URL.

The `posts/1/comments/2` should return 404, if the post id of the comment is not `1`.

You can define scoped params using the `>` greater than sign or famously known as the [breadcrumb sign](https://www.smashingmagazine.com/2009/03/breadcrumbs-in-web-design-examples-and-best-practices/#:~:text=You%20also%20see%20them%20in,the%20page%20links%20beside%20it.)

```ts
Route.get('/posts/:post/comments/:>comment', 'PostsController.show')
```

```ts
import Post from 'App/Models/Post'
import Comment from 'App/Models/Comment'
import { bind } from '@adonisjs/route-model-binding'

export default class PostsController {
  @bind()
  public async show({}, post: Post, comment: Comment) {
    return { post, comment }
  }
}
```

In order for the above example to work, you will have to define the `comments` as a relationship on the `Post` model. The type of the relationship does not matter.

```ts
class Post extends BaseModel {
  @hasMany(() => Comment)
  public comments: HasMany<typeof Comment>
}
```

The name of the relationship is looked up converting the param name to `camelCase`. Both plural and signular forms will be used to find the relationship.

### Customizing relationship lookup
By default, the relationship is fetched using the lookup key of the binded child model. Effectively the following query is executed.

```ts
await parent
  .related('relationship')
  .query()
  .where(lookupKey, value)
  .firstOrFail()
```

However, you can customize the lookup by defining `findRelatedForRequest` method on the model (note, this is not a static method).

```ts
class Post extends BaseModel {
  public findRelatedForRequest(ctx, param, value) {
    /**
     * Have to do this weird dance coz of
     * https://github.com/microsoft/TypeScript/issues/37778
     */
    const self = this as unknown as Post
    const lookupKey = param.lookupKey === '$primaryKey' ? 'id' : param.lookupKey

    if (param.name === 'comment') {
      return self
      .related('comments')
      .query()
      .where(lookupKey, value)
      .firstOrFail()
    }
  }
}
```

## Unbinded params
Many times you will have parameters which are raw values and cannot be tied to a model. In the following example, the `version` is a regular string value and not backed using database.

```ts
Route.get(
  '/api/:version/posts/:post',
  'PostsController.show'
)
```

You can represent the `version` as a string on the controller method and no database lookup will be performed. For example:

```ts
import Post from 'App/Models/Post'
import { bind } from '@adonisjs/route-model-binding'

class PostsController {
  @bind()
  public async show({}, version: string, post: Post) {}
}
```

Since, the route params and the controller method arguments are matched in the same order they are defined, you will always have to typehint all the parameters.
