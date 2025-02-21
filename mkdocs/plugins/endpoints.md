
# &#x1f310; Custom Endpoints

Custom endpoints allow you to extend the REST API offered by the Cat.  
All endpoints are documented directly on your installation under [`localhost:1865/docs`](http://localhost:1865/docs), with usage examples and a playground to try them out.

## How to add a custom endpoint

Let's add a simple endpoint with no input and no auth.  
Add the following to your plugin:

```python
from cat.mad_hatter.decorators import endpoint

@endpoint.get("/new")
def my_endpoint():
    return "meooow"
```

Now open your browser on [`localhost:1865/custom/new`](http://localhost:1865/custom/new), you should see a `meooow` in the page.
The new endpoint also appeared in `/docs` alongside core endpoints, under the `Custom Endpoints` group.  

## Authentication and Authorization via StrayCat

You'll probably want to:

 1. restrict your custom endpoints to requests providing the [correct key or jwt](../production/auth/authentication.md)
 2. access the [user session and main Cat's modules](../framework/cat-components/cheshire_cat/stray_cat.md) from within the endpoint

As an example let's have the endpoint producing a joke:

```python
from cat.mad_hatter.decorators import endpoint
from cat.auth.permissions import check_permissions

@endpoint.get("/joke")
def joke(cat=check_permissions("CONVERSATION", "WRITE")):
    
    # invoking the LLM!
    return cat.llm("Tell me a short joke.")
```

We all know LLMs' jokes are rarely fun, but you can generate a new one every time you access endpoint `GET /custom/joke`.

Notice here we used `cat` as we did in hooks and tools. It is in all cases an instance of [`StrayCat`](./../framework/cat-components/cheshire_cat/stray_cat.md), to let you easily access user session, LLM and most of the functionality the framework can offer.

Utility function `check_permissions` will handle authentication and authorization, both for api keys and jwt, giving in output a `StrayCat` if successful. The function requires you to specify a resource (e.g. `PLUGINS`, `MEMORY`) and a permission (e.g. `READ`, `WRITE`); you can see available resources and permissions in the user manager (admin panel) and in source code under `cat/auth/permissions.py`.  

For simplicity you can write resource and permission as strings, and they will be automatically validated. Under the hood those are treated as enums and you can use those directly:

```python
from cat.mad_hatter.decorators import endpoint
from cat.auth.permissions import AuthResource, AuthPermission, check_permissions

@endpoint.get("/joke")
def joke(cat=check_permissions(AuthResource.CONVERSATION, AuthPermission.WRITE)):
    
    # invoking the LLM!
    return cat.llm("Tell me a short joke.")
```

!!!warning
    If your endpoint function does not have a `cat=check_permissions(...)` argument, the endpoint will be wide open to the web.


## Endpoint input and output

To make your endpoint work with custom data structures, it is a good idea to make [pydantic](https://docs.pydantic.dev/latest/) models describing input and output, so you get code clarity plus automatic validation and documentation.

Let's imagine an endpoint you can call from any client, for example a Javascript frontend or a Rust batch job running in the night on a remote server. Endpoint will receive `topic` and `language` for the joke, and send as output `joke` and `user_id`.

```python
from pydantic import BaseModel

class JokeInput(BaseModel):
    topic: str
    language: str

class JokeOutput(BaseModel):
    joke: str
    user_id: str

@endpoint.post("/topic-joke")
def topic_joke(
    joke_input: JokeInput,
    cat=check_permissions("CONVERSATION", "WRITE"),
) -> JokeOutput:

    joke = cat.llm(f"Tell me a short joke about {joke_input.topic}, in {joke_input.language} language.")
    
    return JokeOutput(
        joke=joke,
        user_id=cat.user_id
    )
```

To use the endpoint send a `POST` request to `/custom/topic-joke` or just use it in the playground under `/docs`. Request payload will be something like:

```json
{
  "topic": "mozzarella",
  "language": "italian"
}
```

Ad the glorious response something like:

```json
{
  "joke": "Perché la mozzarella non va mai in palestra? \n\nPerché ha paura di sciogliersi!",
  "user_id": "user"
}
```

As you can see specifying input and output models gave you automatic validation and automatic documentation. You can avoid typing everything in prototyping stage, but it is a good practice for production.


## Path and Tags

You are free to name the endpoint as you please, using any HTTP verb (`GET`, `POST`, `PUT`, `DELETE`) and deciding in autonomy what the endpoint gets as input and gives as output. You can also customize the endpoint's path: 

```python
@endpoint.get(path="/joke", prefix="/random", tags=["Useful Stuff"])
def random_joke():
    return 42
```

This time the endpoint will be listening on `http://localhost:1865/random/joke` and will be documented in `/docs` in its own `Useful Stuff` group.

## Dependency injection

Being a full blown [FastAPI](https://fastapi.tiangolo.com/) endpoint, you can use any primitive available in FastAPI.  
Here is an example to access directly the network request:

```python
from fastapi import Request

@endpoint.get("/headers")
def send_me_back_the_headers(request: Request):
    return request.headers
```


## Examples

TODO CONTRIBUTIONS ARE WELCOME

