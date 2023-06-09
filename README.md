# Hello

To start your Phoenix server:

- Run `mix setup` to install and setup dependencies
- Start Phoenix endpoint with `mix phx.server` or inside IEx with `iex -S mix phx.server`

Now you can visit [`localhost:4000`](http://localhost:4000) from your browser.

Ready to run in production? Please [check our deployment guides](https://hexdocs.pm/phoenix/deployment.html).

## Learn more

- Official website: https://www.phoenixframework.org/
- Guides: https://hexdocs.pm/phoenix/overview.html
- Docs: https://hexdocs.pm/phoenix
- Forum: https://elixirforum.com/c/phoenix-forum
- Source: https://github.com/phoenixframework/phoenix

# My Notes

Records the learning points by following the Phoenix offical documents

## Components

- Endpoint, the entry point for all http request.
- Router, defines the rules to dispatch request to controllers.
- Controller, defines the view module to render HTML to client.
- Telemetry, collect metrics and send monitoring events of our application.

## From endpoint to views

Use elixir dummy sudo code to represent

```elixir
request
|> endpoint
|> router # It maps http request to one action of a controller
|> controller  # Inside the controller action, we define which template from view to render.
|> template # Either functional component or embed_templates to render the template.
```

## Where to use Plug

- [Endpoint plugs](https://hexdocs.pm/phoenix/plug.html#endpoint-plugs)
  - We could add plugs in endpoint to make it available to all requests.
- [Router plugs](https://hexdocs.pm/phoenix/plug.html#router-plugs)
  - Routes are defined inside scopes.
    - Inside a scope, we could pipe_through multiple pipeline.
  - We add plugs inside the pipeline.
    - Inside a pipeline, there could be multiple plugs.
- [Controller plugs](https://hexdocs.pm/phoenix/plug.html#controller-plugs)
  - Allow us to execute plugs only within certain actions.
  - A good example for showing [how to compose plugs and used in controller](https://hexdocs.pm/phoenix/plug.html#plugs-as-composition).

## Routing

- Use [resources](https://hexdocs.pm/phoenix/routing.html#resources) to define REST for a resource
  - For example, `resources "/users", UserController` will automatically add the following routes for us, a standard matrix of HTTP verbs, paths, and controller actions:
    ```text
    ...
    GET     /users           HelloWeb.UserController :index
    GET     /users/:id/edit  HelloWeb.UserController :edit
    GET     /users/new       HelloWeb.UserController :new
    GET     /users/:id       HelloWeb.UserController :show
    POST    /users           HelloWeb.UserController :create
    PATCH   /users/:id       HelloWeb.UserController :update
    PUT     /users/:id       HelloWeb.UserController :update
    DELETE  /users/:id       HelloWeb.UserController :delete
    ...
    ```
  - We could define [Nested resources](https://hexdocs.pm/phoenix/routing.html#nested-resources)!
- [Verify routes](https://hexdocs.pm/phoenix/routing.html#verified-routes) throughout web layer by using `~p`.

  - It verify the route exists at compile time.

- The purpose of scope is to group similar routes with common path prefix together.

  - For example, we want to group admin related route into a different group, seperate from other actions

    ```elixir
    scope "/", HelloWeb do
      pipe_through :browser

      ...
      resources "/reviews", ReviewController
    end

    scope "/admin", HelloWeb.Admin do
      pipe_through :browser

      resources "/reviews", ReviewController
    end
    ```

  - The above example will create multiple actions for different scope for different prefix route: `/` vs `/admin`.
  - We could define [two scope with the same prefix path](https://hexdocs.pm/phoenix/routing.html#how-to-organize-my-routes).

- [Pipelines](https://hexdocs.pm/phoenix/routing.html#pipelines) are a series of plugs that can be attached to a certain scope.

  - We could create custom pipeline anywhere in the router.
  - For example, [define a new pipeline](https://hexdocs.pm/phoenix/routing.html#creating-new-pipelines) for authentication and add series of plugs to it.
  - Don't remember to attache the pipeline to a scope.

- Use `forward` to [forward]https://hexdocs.pm/phoenix/routing.html#forward() all request with a prefix path to a plug.

## Controllers

- A intermediary modules act between route and view. It is also a plug by itself.
- Controller functions (actions) [naming convension](https://hexdocs.pm/phoenix/controllers.html#actions).

  - Each function always has signature: `(conn, params)`

- There are several ways for controller to render content
  - `text/2`
  - `json/2`, useful for writing APIs.
  - `html/2`
  - `render/3`
    - Controller and view must share the same root name (in our case, it is `Hello`).
    - Use functional component or embed_templates.
    - By default the controller, view module, and templates are collocated together in the same controller directory.
- We could make a controller to support render both html and json by specifying which views to use for each format.

  - In controller, explicitly set the following:
    ```elixir
    plug :put_view, html: HelloWeb.PageHTML, json: HelloWeb.PageJSON
    ```
  - Create view module `HelloWeb.PageJSON` and return json object from template.
  - Edit router to be: `plug :accepts, ["html", "json"]`.

- Setting the content type
  ```elixir
  conn
  |> put_resp_content_type("text/xml")
  |> render(:home, content: some_xml_content)
  ```
- Setting the HTTP Status
  ```elixir
  conn
  |> put_status(202)
  |> render(:home, layout: false)
  ```

### Redirection

- Redirect within application using `~p` sigil.
- Redirect outside URL with full-quanlified path.

## Components and HEEX

- What is function component ?
  Any function that accepts an assigns parameter and returns a HEEx template to be a function component.

## Ecto

- Create schema by using `mix ecto.gen.migration <migration_name>`.
  - This will create a migration file and let us to edit it to fill it with any schema definition.
  - We could also [define schema](https://hexdocs.pm/phoenix/ecto.html#using-the-schema-and-migration-generator) directly by using `mix phx.gen.schema`
- `Changeset` define a pipeline of transformations our data needs to undergo before it will be ready for our application to use.
- `Repo` take care of the finer details of persistence and data querying for us.
  - We pass a changeset to `Repo.insert/2` to insert.
  - We cound also insert data model directly.

## Contexts

- What are contexts ? [Thinking about design](https://hexdocs.pm/phoenix/contexts.html#thinking-about-design)
  - Contexts are dedicated modules that expose and group related functionality.
  - In Phoenix, contexts often encapsulate data access and data validation.
- How to come up a name for context?

  - If you're stuck when trying to come up with a context name when the grouped functionality in your system isn't yet clear, you can simply use the plural form of the resource you're creating.
  - For example, a Products context for managing products. As you grow your application and the parts of your system become clear, you can simply rename the context to a more refined one.

- Helper functions

  - `mix phx.gen.html`
  - `mix phx.gen.json`
  - `mix phx.gen.live`
  - `mix phx.gen.context`

  The differences between `mix phx.gen.context` vs `mix phx.gen.html` is that: `mix phx.gen.context` won't generate web related files for us.

- Example: Create A context for showcasing a product and managing the exhibition of products.

  - Generate context `Catalog`
    ```elixir
    mix phx.gen.html Catalog Product products title:string description:string price:decimal views:integer
    ```
    It generates 4 parts
    - Context module `Hello.Catalog` in `lib/hello/catalog.ex`.
    - Schema module `Hello.Catalog.Product` in `lib/hello/catalog/product.ex`.
    - Web controller and views for `product`:
      - `HelloWeb.ProductController`, controller module in `lib/hello_web/controllers/product_controller.ex`.
      - `HelloWeb.ProductHTML`, view module in `lib/hello_web/controllers/product_html.ex`.
      - Several template files such as`xxx.heex.html` in folder `lib/hello_web/controllers/product_html`.
    - Test related files.
  - Run `mix ecto.migrate`.
  - Edit `lib/hello_web/router.ex to` to include `resources "/products", ProductController`.

- What have learned from above example:

  - Our Phoenix controller is the web interface into our greater application. Our business logic and storage details are decoupled by context.
    - Therefore, the controller talks to`Catelog` context module instead of `Product` schema module.
      - Visit to product in Router <> actions in controller.
      - Do business stuff: controller <> context.
      - Do schema stuff: context <> schema.
    - In other words, from controller's point of view, how product fetching and creation is happening under the hood.

- Example: Add new functions into context.
  - Suppose we want to [tracking product page view count](https://hexdocs.pm/phoenix/contexts.html#adding-catalog-functions).
  - Think of a function that describes what we want to accomplish and be careful the race condition.

### [In-context Relationships](https://hexdocs.pm/phoenix/contexts.html#in-context-relationships)

- How to determine if two resources belong to the same context or not?  
  In general, if you are unsure, you should prefer separate modules (contexts).
- How to add a many to many relationship to an existing resource ?
  For example, a product can have multiple categories, and a category can contain multiple products.

  - Create `Category` by: `mix phx.gen.context Catalog Category categories`.
  - Create relationship table by: `mix ecto.gen.migration create_product_categories`.
    - Define table with foreign_key, index and unique_index.
  - Fill sample data using `priv/repo/seeds.exs`.
  - Modify existing `lib/hello/catalog/product.ex` schema to add many_to_many relationship to categories.
  - Modify existing `lib/hello/catalog.ex` context.

    - Modify product to repload categories.
    - When change product

      - When fetch product, also load categories from category_ids.
      - For change a product: Preload categories and do changeset, then put_assoc

        ```elixir
        def change_product(%Product{} = product, attrs \\ %{}) do
          # Product.changeset(product, attrs)
          categories = list_categories_by_id(attrs["category_ids"])

          product
          |> Repo.preload(:categories)
          |> Product.changeset(attrs)
          |> Ecto.Changeset.put_assoc(:categories, categories)
        end
        ```

  - Adding the category input to the product form.
    - Create new function component `lib/hello_web/controllers/product_html.ex`.
    - Use function component in `lib/hello_web/controllers/product_html/product_form.html.heex` for selecting categories in product form.
    - Modify `lib/hello_web/controllers/product_html/show.html.heex` to show categories for a product.

### [Cross-context dependencies](https://hexdocs.pm/phoenix/contexts.html#cross-context-dependencies)

#### Create ShoppingCart Context

In order to properly track products that have been added to a user's cart, we build the "carting products from the catalog" feature.

- Generate `Cart` in `ShoppingCart` context

  ```sh
  mix phx.gen.context ShoppingCart Cart carts user_uuid:uuid:unique
  ```

- Generate `CartItem` in `ShoppingCart` context.
  ```sh
  mix phx.gen.context ShoppingCart CartItem cart_items \
  cart_id:references:carts product_id:references:products \
  price_when_carted:decimal quantity:integer
  ```
- Do further modification to the generated migration file `create_cart_items.exs` to enhance:

  - price precision and scale
  - foreign_key's on_delete operation
  - create unique index from two index's combination

- Run `mix ecto.migrate`.

#### Do cross-context data

Resolve the dependency between resources

- Setup association for schemas which has dependencies for each other. In our case, `ShoppingCart` context have a data dependency on the `Catalog` context.
  - One solution is to use database joins to fetch the dependent data. (Our choice).
  - Another one is to expose APIs on the `Catalog` context to allow us to fetch product data for use in the `ShoppingCart` system.
- Notice `has_many` vs `belong_to`
  - `Cart` has many `CartItem`.
  - `CartItem` belong to `Cart`,
  - `CartItem` belong to `Product`
- Notice: we add `CartItem` instead of `Product` into our `Cart`.
  - Between different contexts, they indicates the resource we use is different and should be treated differently.
  - For example, `Product` is used in `Catelog` but could not be used directly in `Cart`.

#### Adding Shopping Cart functions

See: [Adding Shopping Cart functions](https://hexdocs.pm/phoenix/contexts.html#adding-shopping-cart-functions)

- Ensure every user of our application is granted a cart if one does not yet exist.
  - Make application support user session (as `Plug`)
    - In connection, we need to assign `user_uuid`. We could get it from session.
    - If there is no `user_uuid`, we create new one, put it into session and assign it into connection.
    - The goal is to ensure there is a current user in the connection.
  - Each connection has one user <==> one cart (as `Plug`).
    - From connection's `current_uuid` (guaranteed from above user session `Plug`), get its corresponding cart.
    - If there is a one, assign it to connection.
    - Otherwise, create a new one for `current_uuid` and assign it to connection.
    - The goal is to ensure there is cart in the connection.
- Allow user add items to their cart, update item quantities, calculate cart totals.
  - Modify router for `cart_item` and `cart`, and create corresponding controllers.
  - `CartController`
  - `CartItemController`, for features related with `cart_item`:
    - Add item to cart from a `product_id`.
    - Remove item from cart from `product_id`
    - The related resources include `Product`, `Cart`, `CartItem`.
  - HTML part
    - "Add to cart" button on the product show page.

### [Adding an Orders context](https://hexdocs.pm/phoenix/contexts.html#adding-an-orders-context)

Order of busniess is to allow the user to initiate the checkout process.

- Allow a user to add a back-ordered item to their cart.
- Could not allow an order with on item to be completed.
- Capture point-in-time product information when an order is completed.

#### Generate `Orders` context

Generate `Order` in `Orders` context

```sh
mix phx.gen.context Orders Order orders user_uuid:uuid totoal_price:decimal
```

Modify the "xxx_create_orders.exs" file to modify the schema to better suit our needs. \
Such as `precision: 15, scale: 6`.

Generate `LineItem` in `Orders` context. \
This will capture the price of each product at payment transaction time.

```sh
mix phx.gen.context Orders LineItem order_line_items \
price:decimal quantity:integer \
order_id:references:orders product_id:references:products
```

Modify the "xxx_create_order_line_items.exs" to enhance the schema with better control on field.

With `create table` migration finished. Now, it is time to edit the corresponding schema file (model) to add associations between them.

To build associations we use `belongs_to` and `has_many`.

- `belongs_to` is used to represent a "foreign_key". \
  For example, in schema `order_line_items` (LineItem.ex), we use `belongs_to :order, Hello.Orders.Order` to indicate a foreign_key to an `Order`.
- `has_many` is used to build back tracing to fill an association. \
  For example, in schema `orders` (Order.ex), we use `has_many :line_items, Hello.Orders.LineItem` to make an `Order` could be filled with association `line_items`.

#### Modify router to include resouce for orders

```elixir
resources "/orders", OrderController, only: [:create, :show]
```

#### Create controller "order"

- To complete an order, our cart page can issue a POST to the `OrderController.create` action.
- To complete an order:
  - A new order record must be persisted with the total price of the order.
  - All items in the cart must be transformed into new order line items records with quantity and point-in-time product price information
  - After successful order insert (and eventual payment), items must be pruned from the cart.

#### Create view

- Show a complete order
- Add "complete order" button to cart page to allow completing an order.

## [JSON and APIs](https://hexdocs.pm/phoenix/json_and_apis.html)

In this part, we will build Web APIs to store our favorite links.

### The JSON API

- Use `mix phx.gen.json` to generate scaffold our API infrastructure.

  ```
  mix phx.gen.json Urls Url urls link:string title:string
  ```

  It generates

  - controller -- `url_controller.ex`
  - context -- `urls.ex`
  - schema -- `url.ex`
  - test files

### Rendering JSON

- In view, it is similar with how HTML is rendered except.
  - For example, it is `UrlJSON` instead of `UrlHTML`.
- Our JSON view converts our complex data into simple Elixir data-structures

### [Action fallback](https://hexdocs.pm/phoenix/json_and_apis.html#action-fallback)

- We could create a controller which dedicates to handle error message from connection. Then, in other controller, use it as action fallback
- For example we have the following fallback controller

  ```elixir
  defmodule HelloWeb.MyFallbackController do
    use Phoenix.Controller

    def call(conn, {:error, :not_found}) do
      conn
      |> put_status(:not_found)
      |> put_view(json: HelloWeb.ErrorJSON)
      |> render(:"404")
    end

    def call(conn, {:error, :unauthorized}) do
      conn
      |> put_status(403)
      |> put_view(json: HelloWeb.ErrorJSON)
      |> render(:"403")
    end
  end
  ```

- We now could use it in other controller as

  ```elixir
  defmodule HelloWeb.MyController do
    use Phoenix.Controller

    action_fallback HelloWeb.MyFallbackController

    def show(conn, %{"id" => id}, current_user) do
      with {:ok, post} <- fetch_post(id),
          :ok <- authorize_user(current_user, :view, post) do
        render(conn, :show, post: post)
      end
    end
  end
  ```

- See [FallbackController and ChangesetJSON](https://hexdocs.pm/phoenix/json_and_apis.html#fallbackcontroller-and-changesetjson) for more details.

## Other references

### About Ecto

- Ecto, they are not tied to the database, and they can be used to map data from and to any source, which makes it a general and useful data structure for tracking field changes, perform validations, and generate error messages.
- `%Ecto.Changeset{}` is a good choice to model the data changes between your contexts and your web layer - regardless if you are talking to an API or the database.
- [Understanding Associations in Elixir's Ecto](https://blog.appsignal.com/2020/11/10/understanding-associations-in-elixir-ecto.html)

## Troubleshooting

- How to prevent vscode automatically add parenthese?
  This is especially annoying for some code, such as Plug related.
  - Solution: seems [no simple solution](https://github.com/elixir-lang/elixir/issues/8165).
