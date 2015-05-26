[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/neomerx/json-api/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/neomerx/json-api/?branch=master)
[![Code Coverage](https://scrutinizer-ci.com/g/neomerx/json-api/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/neomerx/json-api/?branch=master)
[![Build Status](https://travis-ci.org/neomerx/json-api.svg?branch=master)](https://travis-ci.org/neomerx/json-api)
[![HHVM](https://img.shields.io/hhvm/neomerx/json-api.svg)](https://travis-ci.org/neomerx/json-api)
[![License](https://img.shields.io/packagist/l/neomerx/json-api.svg)](https://packagist.org/packages/neomerx/json-api)

## Description

<img src="http://jsonapi.org/images/jsonapi.png" alt="JSON API logo" title="JSON API" align="right" width="415" height="130" />

A good API is one of most effective ways to improve the experience for your clients. Standardized approaches for data formats and communication protocols increase productivity and make integration between applications smooth.

This framework agnostic package implements [JSON API](http://jsonapi.org/) specification version RC4 (which is planned to be released on May 28, 2015 as v1.0) and helps focusing on core application functionality rather than on protocol implementation. It supports document structure, errors and data fetching as described in [JSON API Format](http://jsonapi.org/format/). As it is designed to stay framework agnostic for practical usage it requires framework integration. If you are looking for quick start application consider one of the following
- [Limoncello Collins](https://github.com/neomerx/limoncello-collins) is a pre-configured Laravel-based quick start application.
- [Limoncello Shot](https://github.com/neomerx/limoncello-shot) is a pre-configured Lumen-based quick start application.

These applications use [Limoncello](https://github.com/neomerx/limoncello) as integration level with Symfony based projects.

Encoding fully support specification. In particular

* Resource attributes and complex attributes
* Compound documents with included resources
* Circular resource references
* Meta information for document, resources and link objects
* Link objects (including links as references, links to null and empty arrays)
* Sparse fieldset filter rules
* Pagination links
* Errors

The package covers all the complexity of parsing and checking HTTP request parameters and headers. For instance it helps to correctly respond with ```Unsupported Media Type``` (HTTP code 415) and ```Not Acceptable``` (HTTP code 406) to invalid requests. You don't need to manually validate all input parameters on every request. You can configure what parameters are supported by your services and this package will check incoming requests automatically. It greatly simplifies API development. All parameters from the specification are supported

* Parsing HTTP ```Accept``` and ```Content-Type``` headers in accordance with RFC 2616
* Inclusion of related resources
* Sparse fields
* Sorting
* Pagination
* Filtering

## Install

Via Composer

```
$ composer require neomerx/json-api ~0.4
```

## Usage

### Basic usage

Assuming you've got an ```$author``` of type ```\Author``` you can encode it to JSON API as simple as this

```php
$encoder = Encoder::instance([
    '\Author' => '\AuthorSchema',
], new JsonEncodeOptions(JSON_PRETTY_PRINT));

echo $encoder->encode($author) . PHP_EOL;
```

will output

```json
{
    "data": {
        "type": "people",
        "id": "123",
        "attributes": {
            "first_name": "John",
            "last_name": "Dow"
        },
        "links": {
            "self": "http:\/\/example.com\/people\/123"
        }
    }
}
```

The ```AuthorSchema``` provides information about resource's fields and might look like

```php
class AuthorSchema extends SchemaProvider
{
    protected $resourceType = 'people';
    protected $baseSelfUrl  = 'http://example.com/people/';

    public function getId($author)
    {
        /** @var Author $author */
        return $author->authorId;
    }

    public function getAttributes($author)
    {
        /** @var Author $author */
        return [
            'first_name' => $author->firstName,
            'last_name'  => $author->lastName,
        ];
    }
}
```

### Object hierarchy with included objects

```php
// '::class' constant (PHP 5.5 and above) convenient for classes in namespaces 
$encoder  = Encoder::instance([
    Author::class  => AuthorSchema::class,
    Comment::class => CommentSchema::class,
    Post::class    => PostSchema::class,
    Site::class    => SiteSchema::class
], new JsonEncodeOptions(JSON_PRETTY_PRINT));

echo $encoder->encode($site) . PHP_EOL;
```

will output

```json
{
    "data": {
        "type": "sites",
        "id": "1",
        "attributes": {
            "name": "JSON API Samples"
        },
        "relationships": {
            "posts": {
                "data": { "type": "posts", "id": "321" }
            }
        },
        "links": {
            "self": "http:\/\/example.com\/sites\/1"
        }
    },
    "included": [
        {
            "type": "people",
            "id": "123",
            "attributes": {
                "first_name": "John",
                "last_name": "Dow"
            }
        },
        {
            "type": "comments",
            "id": "456",
            "attributes": {
                "body": "Included objects work as easy as basic ones"
            },
            "relationships": {
                "author": {
                    "data": { "type": "people", "id": "123" }
                }
            },
            "links": {
                "self": "http:\/\/example.com\/comments\/456"
            }
        },
        {
            "type": "comments",
            "id": "789",
            "attributes": {
                "body": "Let's try!"
            },
            "relationships": {
                "author": {
                    "data": { "type": "people", "id": "123" }
                }
            },
            "links": {
                "self": "http:\/\/example.com\/comments\/789"
            }
        },
        {
            "type": "posts",
            "id": "321",
            "attributes": {
                "title": "Included objects",
                "body": "Yes, it is supported"
            },
            "relationships": {
                "author": {
                    "data": { "type": "people", "id": "123" }
                },
                "comments": {
                    "data": [
                        { "type": "comments", "id": "456" },
                        { "type": "comments", "id": "789" }
                    ]
                }
            }
        }
    ]
}
```

### Sparse and fields sets

Output result could be filtered by included relations and object attributes.

```php
$options  = new EncodingParameters(
    ['posts.author'], // Paths to be included
    [
        // Attributes and links that should be shown
        'sites'  => ['name'],
        'people' => ['first_name'],
    ]
);
$encoder  = Encoder::instance([
    Author::class  => AuthorSchema::class,
    Comment::class => CommentSchema::class,
    Post::class    => PostSchema::class,
    Site::class    => SiteSchema::class
], new JsonEncodeOptions(JSON_PRETTY_PRINT));

echo $encoder->encode($site, null, null, $options) . PHP_EOL;
```

Note that ```sites```, ```posts``` and ```people``` attributes are filtered.

```json
{
    "data": {
        "type": "sites",
        "id": "1",
        "attributes": {
            "name": "JSON API Samples"
        },
        "links": {
            "self": "http:\/\/example.com\/sites\/1"
        }
    },
    "included": [
        {
            "type": "people",
            "id": "123",
            "attributes": {
                "first_name": "John"
            }
        },
        {
            "type": "posts",
            "id": "321"
        }
    ]
}
```

### View customization

You can fully customize how your output result will look like

* Add links to document.
* Add meta to document, resource objects or link objects.
* Show object links as references.
* Show/hide ```self```, ```related```, ```links```, ```meta``` for each resource type individually for both main and included objects and links.
* Specify what links should place resources to ```included``` section for each resource type independently.

### Sample source code

The full sample source code with performance test could be found [here](sample/)

## Advanced usage

### Link resources

Adding links to a resource is easy. You only need to add ```getLinks``` method to resource schema

```php
class PostSchema extends SchemaProvider
{
    ...
    public function getLinks($post)
    {
        /** @var Post $post */
        return [
            'author'   => [self::DATA => $post->author],
            'comments' => [self::DATA => $post->comments],
        ];
    }
    ...
}
```

Apart from ```self::DATA``` the following keys are supported

* ```self::SHOW_AS_REF``` (bool) - if this key is set to ```true``` the link will be rendered as a reference. Default ```false```.
* ```self::SHOW_META``` (bool) - if resource meta information should be added to output link description. Default ```false```.
* ```self::SHOW_LINKAGE``` (bool) - if linkage information should be added to output link description. Default ```true```.
* ```self::SHOW_SELF``` (bool) - if link ```self``` URL should be added to output link description. Default ```false```.
* ```self::SHOW_RELATED``` (bool) - if link ```related``` URL should be added to output link description. Default ```false```.

### Include linked resources

By default linked resources will not be included to ```included``` output section. However you can change this behaviour by defining 'include paths'. Nested links should be separated by a dot ('.'). For instance

```php
class PostSchema extends SchemaProvider
{
    ...
    public function getIncludePaths()
    {
        return [
            'posts',
            'posts.author',
            'posts.comments',
        ];
    }
    ...
}
```

If you specify ```EncodingParametersInterface``` with 'included paths' set to ```Encoder::encode``` method it will override include paths in the ```Schema```.

### Provide resource's meta information

A ```getMeta``` method should be added to schema

```php
class PostSchema extends SchemaProvider
{
    ...
    public function getMeta($post)
    {
        /** @var Post $post */
        return [
            ...
        ];
    }
    ...
}
```

### Show/Hide 'self' or 'meta' for resource

You can set it up in resource schema

```php
class YourSchema extends SchemaProvider
{
    ...
    protected $isShowSelf = true;
    protected $isShowMeta = false;
    ...
}
```

### Show/Hide 'self' or 'meta' for included resource


```php
class YourSchema extends SchemaProvider
{
    ...
    protected $isShowSelfInIncluded  = false;
    protected $isShowLinksInIncluded = false;
    protected $isShowMetaInIncluded  = false;
    ...
}
```

### Show top level links and meta information

```php
$author = Author::instance('123', 'John', 'Dow');
$meta   = [
    "copyright" => "Copyright 2015 Example Corp.",
    "authors"   => [
        "Yehuda Katz",
        "Steve Klabnik",
        "Dan Gebhardt"
    ]
];
$links  = new DocumentLinks(
    null,
    'http://example.com/people?first',
    'http://example.com/people?last',
    'http://example.com/people?prev',
    'http://example.com/people?next'
);

$encoder  = Encoder::instance([
    Author::class  => AuthorSchema::class,
    Comment::class => CommentSchema::class,
    Post::class    => PostSchema::class,
    Site::class    => SiteSchema::class
], new JsonEncodeOptions(JSON_PRETTY_PRINT));

echo $encoder->encode($author, $links, $meta) . PHP_EOL;
```

will output

```json
{
    "meta": {
        "copyright": "Copyright 2015 Example Corp.",
        "authors": [
            "Yehuda Katz",
            "Steve Klabnik",
            "Dan Gebhardt"
        ]
    },
    "links": {
        "first": "http:\/\/example.com\/people?first",
        "last": "http:\/\/example.com\/people?last",
        "prev": "http:\/\/example.com\/people?prev",
        "next": "http:\/\/example.com\/people?next"
    },
    "data": {
        "type": "people",
        "id": "123",
        "attributes": {
            "first_name": "John",
            "last_name": "Dow"
        },
        "links": {
            "self": "http:\/\/example.com\/people\/123"
        }
    }
}
```

### Dynamic Schemas

Encoder supports dynamic schemas. Instead of schema class name you can specify ```Closure``` which will be invoked on schema creation. This feature could be used for setting up schemas based on configuration settings, environment variables, user input and etc. 

```php
$schemaClosure = function () {
    $schema = new CommentSchema(..., ..., ...);
    return $schema;
};

$encoder = Encoder::instance([
    Author::class  => AuthorSchema::class,
    Comment::class => $schemaClosure,
    Post::class    => PostSchema::class,
    Site::class    => SiteSchema::class
]);
```

## Your API

Are you planning to add JSON API and need help? We'd love to talk to you [sales@neomerx.com](mailto:sales@neomerx.com).

## Contributing

If you have spotted any specification changes that are not reflected in this package please post an [issue](https://github.com/neomerx/json-api/issues). Pull requests for improving documentation and code are welcome.

Thank you for your support :star:.

## Testing

``` bash
$ phpunit
```

## Versioning

This package is using [Semantic Versioning](http://semver.org/).

## Credits

- [Neomerx](https://github.com/neomerx)
- [All Contributors](../../contributors)

## License

Apache License (Version 2.0). Please see [License File](LICENSE) for more information.
