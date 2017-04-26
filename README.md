# Designing An API With RAML 1.0
*From The Initial Idea to The Actual Implementation*

**requirements**: [Node.js](https://nodejs.org/en/download/), [HTTPie](https://github.com/jakubroztocil/httpie), [jq](https://github.com/stedolan/jq)

## Where To Start?

When I came across *[SWAPI: The Star Wars API](http://swapi.co/documentation)*, I thought it would be fun to try to reproduce that API. It provides JSON schemas for all its resources which means that I can use those schemas as a base to building my own API without much effort.

```sh
$ http -b swapi.co/api/starships/schema
```

## Converting JSON Schemas To RAML Data Types

[`ramldt2jsonschema`](https://github.com/raml-org/ramldt2jsonschema) is a CLI tool that converts JSON schemas to RAML data types, and vice-versa. After installing it, there are two commands available: `js2dt` and `dt2js`.

```sh
$ npm install -g ramldt2jsonschema
$ js2dt schema.json Starship
```

Once I export the `Starship` type, I can refer to it from within any RAML spec. I could have stuck to using this JSON schema directly - since RAML supports JSON schemas - but I would have had to write a second schema for my `GET /starships/` method since it returns a list of objects as opposed to a single object. What's great about that RAML data type is that I can use it from both methods that return a single object, such as `POST /starships/`, and methods that return lists of objects, such as `GET /starships/`.

## Mocking Using Example Data

Since SWAPI returns very detailed data, I can use that data to create examples for my RAML spec. I need an example for my `GET /starships/` method and another example for my `POST /starships/` method. I'll grab them using jq.
```sh
$ http -b swapi.co/api/starships/ | jq '.results[0:2]'
$ http -b swapi.co/api/starships/9/ | jq .
```

Now that I have examples, I can refer to those examples from within my RAML spec and then use [`osprey-mock-service`](https://github.com/mulesoft-labs/osprey-mock-service) to serve those examples as an API mock service. `osprey-mock-service` uses [Osprey](https://github.com/mulesoft/osprey), an HTTP request middleware for Node.js, so it will also validate my POST requests. 
```sh
$ npm install -g osprey-mock-service
$ osprey-mock-service -f api.raml -p 8080
Mock service running at http://localhost:8080
```

We can use this mock service to start testing requests defined in our RAML spec. In a separate window, I can test my two methods as if I already had implemented my API.
```sh
$ http :8080/starships/
HTTP/1.1 200 OK
(...)

[
    {
        "MGLT": "70",
        "cargo_capacity": "180000",
        "consumables": "1 month",
        "cost_in_credits": "240000",
        "created": "2014-12-10T15:48:00.586000Z",
        "crew": "5",
        "edited": "2014-12-22T17:35:44.431407Z",
        "hyperdrive_rating": "1.0",
        "length": "38",
        "manufacturer": "Sienar Fleet Systems, Cyngus Spaceworks",
        "max_atmosphering_speed": "1000",
        "model": "Sentinel-class landing craft",
        "name": "Sentinel-class landing craft",
        "passengers": "75",
        "starship_class": "landing craft",
        "url": "http://swapi.co/api/starships/5/"
    },
    {
        "MGLT": "10",
        "cargo_capacity": "1000000000000",
        "consumables": "3 years",
        "cost_in_credits": "1000000000000",
        "created": "2014-12-10T16:36:50.509000Z",
        "crew": "342953",
        "edited": "2014-12-22T17:35:44.452589Z",
        "hyperdrive_rating": "4.0",
        "length": "120000",
        "manufacturer": "Imperial Department of Military Research, Sienar Fleet Systems",
        "max_atmosphering_speed": "n/a",
        "model": "DS-1 Orbital Battle Station",
        "name": "Death Star",
        "passengers": "843342",
        "starship_class": "Deep Space Mobile Battlestation",
        "url": "http://swapi.co/api/starships/9/"
    }
]
$ http POST :8080/starships/ foo=bar
HTTP/1.1 400 Bad Request
(...)

{
    "errors": [
        {
            "dataPath": "MGLT",
            "keyword": "required",
            "message": "Missing required property: MGLT",
            "schema": true,
            "type": "json"
        },
        {
            "dataPath": "passengers",
            "keyword": "required",
            "message": "Missing required property: passengers",
            "schema": true,
            "type": "json"
        },
        {
            "dataPath": "consumables",
            "keyword": "required",
            "message": "Missing required property: consumables",
            "schema": true,
            "type": "json"
        },
        (...)
}
```

## Generating Documentation

I can also generate some HTML documentation using my RAML spec and [`raml2html`](https://github.com/raml2html/raml2html). This can be very useful during development and obviously later-on when people start consuming my API. After generating the html with `raml2html`, I will be able to open the outputed html file in my favorite web browser to preview a web-friendly version of my API spec.
```sh
$ npm install -g raml2html
$ raml2html api.raml > index.html
```

## Ready to Start Implementing?

Now that I speced-out my API in a RAML file, that I visualized that it works as intended, I can generate annotated Java stubs using [`raml-for-jax-rs`](https://github.com/mulesoft-labs/raml-for-jax-rs).
```sh
$ http -d https://repository-master.mulesoft.org/nexus/content/repositories/releases/org/raml/raml-to-jaxrs-cli/2.0.0/raml-to-jaxrs-cli-2.0.0-jar-with-dependencies.jar
$ java -jar raml-to-jaxrs-cli-2.0.0-jar-with-dependencies.jar -g jackson,javadoc,jsr303 -d ./src/ -r co.swapi api.raml
```

This will generate Java source code inside a `src/` folder annotated with `jackson`, `javadoc`, and `jsr303` annotations.
