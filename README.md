# Testing your APIs in a new way

My experience is that almost every API tests follow the same workflow. 
Fixtures are loaded in setupBeforeClass or maybe in a bootstrap file of
your test framework. A request is send and the response should match.

After I did stumble over the test cases in the [rectorphp](https://github.com/rectorphp/rector-src/pull/1668/files).
I was thinking about how I could adopt this also for APIs to get my API
tests in a format which are framework independent.

## Creating a framework independent format

While thinking about framework independent I first was trying around
with a json format like:

```json
{
    "request": {
        "method": "GET",
        "uri": "/api/examples/1",
        "headers": {
            "Accept": "application/json"
        }
    },
    "response": {
        "method": "GET",
        "uri": "/api/examples/1",
        "content": "/api/examples/1",
        "headers": {
            "Accept": "application/json"
        }
    }
}
```

But I think that that format is not very readable and json has the
disadvantage that we can not add any comment to it.

So I did think about using markdown flavoured format for this:

~~~md
# Request

```http request
POST /api/examples
Accept: application/json

{
    "title": "Test"
}
```

---

# Response

```http request
HTTP/1.1 200 OK
Content-Type: application/json

{
    "id": "@int@",
    "title": "Test"
}
```
~~~

Why at first place the format is really great because it matches
the HTTP protocal it has one downside and it is the autocomplete
and code highlighting of the JSON in the IDE. So to fix that one I
split the content into an own block:

~~~
# Request

```http request
POST /api/examples
Accept: application/json
```

```json
{
    "title": "Test"
}
```

---

# Response

```http request
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
    "id": "1",
    "title": "Test"
}
```
~~~

Now we have the format how we want to write tests.

## Creating a basic test case

To create our basic test case we first need a method to read our markdown files.
We will use here the [symfony/finder](https://symfony.com/doc/current/components/finder.html)
component to read the only files we need.

```php
<?php

namespace App\Tests\Functional;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
use Symfony\Component\Finder\Finder;

abstract class AbstractApiTest extends WebTestCase
{
    /**
     * @return \Generator<\SplFileInfo>
     */
    protected function yieldFilesFromDirectory(string $directory): \Generator
    {
        $finder = new Finder();
        $finder->in($directory)
            ->ignoreVCS(true)
            ->files()
            ->name('*.md');

        foreach ($finder as $file) {
            if ($file instanceof \SplFileInfo) {
                yield [$file];
            }
        }
    }
}
```

Now we need to add also a method to parse the markdown file and make our expected
assertions by them:

```php
protected function doTestFileInfo(\SplFileInfo $fileInfo): void
{
    // arrange
    $content = \file_get_contents($fileInfo->getPathname());

    [$input, $output] = \explode("\n---\n", $content);

    [$method, $uri, $headers, $content] = $this->parseRequest($input);
    [$expectedServerProtocol, $expectedStatusCode, $expectedHeaders, $expectedContent] = $this->parseResponse($output);

    /** @var array<string, string> $server */
    $server = [];
    foreach ($headers as $key => $value) {
        $servers['HTTP_' . strtoupper(str_replace('-', '_', $key))] = $value;
    }

    // act
    $client = $this->createClient();
    $client->request($method, $uri, [], [], $server, $content);

    // assert
    $response = $client->getResponse();

    $this->assertSame($expectedServerProtocol, $response->getProtocolVersion());
    $this->assertSame($expectedStatusCode, $response->getStatusCode());
    foreach ($expectedHeaders as $headerName => $expectedHeaderValue) {
        $this->assertSame($expectedHeaderValue, $response->headers->get($headerName));
    }
    $this->assertSame($expectedContent, $response->getContent());
}
```

The markdown file I did parse quickly with a regex to get out the information
I needed.

<details>
    <summary>`parseRequest`</summary>

```php
/**
 * @return array{
 *     0: string,
 *     1: string,
 *     2: array<string, string>,
 *     3: string,
 * }
 */
private function parseRequest(string $input): array
{
    preg_match('/```http request\n(\w+) (.+)\n([^```]*)```([^```]*```\w+\n([^```]*)```)?/', $input, $matches);
    $method = $matches[1];
    $uri = $matches[2];
    $headerString = $matches[3];
    $content = $matches[5] ?? '';

    $headers = [];
    $headerParts = array_filter(explode("\n", $headerString));
    foreach ($headerParts as $headerPart) {
        [$headerName, $headerValue] = \explode(':', $headerPart, 2);
        $headers[trim($headerName)] = trim($headerValue);
    }

    return [$method, $uri, $headers, $content];
}
```    

</details>

<details>
    <summary>`parseResponse`</summary>

```php
/**
 * @return array{
 *     0: string,
 *     1: int,
 *     2: array<string, string>,
 *     3: string,
 * }
 */
private function parseResponse(string $output): array
{
    preg_match('/```http request\n\w+\/(\d+.\d+) (\d+) \w+\n([^```]*)```([^```]*```\w+\n([^```]*)```)?/', $output, $matches);
    $serverProtocol = $matches[1];
    $statusCode = (int) $matches[2];
    $headerString = $matches[3];
    $content = $matches[5] ?? '';

    $headers = [];
    $headerParts = array_filter(explode("\n", $headerString));
    foreach ($headerParts as $headerPart) {
        [$headerName, $headerValue] = \explode(':', $headerPart, 2);
        $headers[trim($headerName)] = trim($headerValue);
    }

    return [$serverProtocol, $statusCode, $headers, $content];
}
```    

</details>

Now we can use our `AbstractApiTest` in our own Test by testing simple Rest endpoint:

```php
<?php

namespace App\Tests\Functional\Controller;

use App\Tests\Functional\AbstractApiTest;

class ExampleControllerTest extends AbstractApiTest
{
    /**
     * @dataProvider provideData()
     */
    public function testFixtures(\SplFileInfo $fileInfo): void
    {
        $this->doTestFileInfo($fileInfo);
    }

    /**
     * @return \Generator<\SplFileInfo>
     */
    public function provideData(): \Generator
    {
        return $this->yieldFilesFromDirectory(__DIR__ . '/fixtures');
    }
}
```

In the testMethod itself or in the `setupBeforeClass` class you even could still
make sure that you are loading some database fixtures which are required for your test case.

## Fix problem with dynamic response content

While it is hard when you example work with date times,
auto generated ids to exact match the response against a json. 
There is a solution for this by using the [coduo php matcher](https://github.com/coduo/php-matcher).

This library allows you to write something like:

```json
{
    "id": "@integer@",
    "text": "@string@.startsWith('Test')"
}
```

To use expression or type matching instead of exact matching.

So instead of:

```php
$this->assertSame($expectedContent, $response->getContent());
```

```php
$this->assertMatchesPattern($expectedContent, $response->getContent());
```

From the `Coduo\PHPMatcher\PHPUnit\PHPMatcherAssertions` Trait.

This way we can use [all coduo patterns](https://github.com/coduo/php-matcher#available-patterns)
inside our response content.

## Optimizing the output format in PHPUnit

I personally prefer to use the `nunomaduro\collision` package
as a printer for my PHPUnit tests. As it output the tests 
in a nice readable way:

![Picture of default output](pictures/output-default.png)

Still it does not indicate here a lot but with a little change in our
API Testcase by using a key in our test generator this way:

```diff
-yield [$file];
+yield \str_replace(\getcwd() . '/', '', $file->getPathname()) => [$file];
```

It will output us the test in the following format:

![Picture of collision output with set key](pictures/output-formatted.png)

Now the nice thing is as we are using the path from `getcwd()` (current directory)
we can directly in our terminal click on the `.md` file path to get to that
test cases and adopt it to our needs.

## Conclusion

With the usage of a more general format we did got ride of a lot of boilerplate
code for our api tests. With the usage of markdown files we even make it possible
to better document our test cases as we could add test for them.
Also I like to use the response and request format of the official http standard.

The tests could also be even ported to another framework if the framework of your
application is changing or even to another language, there you would just need
to reimplement your base test case again.

If you want to test it yourself feel free to clone this repository and run its tests:

```bash
git clone https://github.com/alexander-schranz/markdown-based-api-testing
composer install
vendor/bin/phpunit
```

Let me know what you think about this way of testing your api. Maybe you know
exist API test frameworks which work the similar way.

Attend the discussion about this on [Twitter](#TODO).