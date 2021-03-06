# Unit Testing a zend-mvc application

> ### TODO
>
> - Remove section on installing phpunit, and replace with installing zend-test.
> - Update assertions against controller names; they're the same as the class now.
> - Try the tests and confirm they work.

A solid unit test suite is essential for ongoing development in large projects,
especially those with many people involved. Going back and manually testing
every individual component of an application after every change is impractical.
Your unit tests will help alleviate that by automatically testing your
application's components and alerting you when something is not working the same
way it was when you wrote your tests.

This tutorial is written in the hopes of showing how to test different parts of
a zend-mvc application. As such, this tutorial will use the application written
in the [getting started user guide](user-guide/overview.md). It is in no way a
guide to unit testing in general, but is here only to help overcome the initial
hurdles in writing unit tests for zend-mvc applications.

It is recommended to have at least a basic understanding of unit tests,
assertions and mocks.

zend-test, which provides testing integration for zend-mvc, uses
[PHPUnit](http://phpunit.de/). This tutorial assumes that you already have
PHPUnit from either a version 4 or version 5 series installed; version 5 is
required if you are using PHP 7.

## Setting up phpunit to use composer's autoload.php

If you used composer to generate an `autoload.php` file for you, per the
skeleton application, then you need to use a phpunit binary installed by
composer. You can add this as a development dependency using composer itself:

```bash
$ composer require --dev phpunit/phpunit
```

The above command will update your `composer.json` file and perform an update
for you, which will also setup autoloading rules.

## Setting up the tests directory

As zend-mvc applications are built from modules that should be
standalone blocks of an application, we don't test the application in it's
entirety, but module by module.

We will demonstrate setting up the minimum requirements to test a module, the
`Album` module we wrote in the user guide, which then can be used as a base
for testing any other module.

Start by creating a directory called `test` in `zf2-tutorial\module\Album` with
the following subdirectories:

```text
zf2-tutorial/
    /module
        /Album
            /test
                /Controller
```

Additionally, add an `autoload-dev` rule in your `composer.json`:

```json
"autoload-dev": {
    "psr-4": {
        "AlbumTest\\": "module/Album/test/"
    }
}
```

When done, run:

```bash
$ composer dump-autoload
```

The structure of the `test` directory matches exactly with that of the module's
source files, and it will allow you to keep your tests well-organized and easy
to find.

## Bootstrapping your tests

Next, create a file called `phpunit.xml.dist` under
`zf2-tutorial/module/Album/test`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit bootstrap="Bootstrap.php" colors="true">
    <testsuites>
        <testsuite name="zf2tutorial">
            <directory>.</directory>
        </testsuite>
    </testsuites>
</phpunit>
```

Now create a file called `Bootstrap.php`, also under `zf2-tutorial/module/Album/test`:

```php
<?php
namespace AlbumTest;

use Zend\Loader\AutoloaderFactory;
use Zend\Mvc\Service\ServiceManagerConfig;
use Zend\ServiceManager\ServiceManager;
use RuntimeException;

error_reporting(E_ALL | E_STRICT);
chdir(__DIR__);

/**
 * Test bootstrap, for setting up autoloading
 */
class Bootstrap
{
    protected static $serviceManager;

    public static function init()
    {
        $zf2ModulePaths = array(dirname(dirname(__DIR__)));
        if (($path = static::findParentPath('vendor'))) {
            $zf2ModulePaths[] = $path;
        }
        if (($path = static::findParentPath('module')) !== $zf2ModulePaths[0]) {
            $zf2ModulePaths[] = $path;
        }

        static::initAutoloader();

        // Use ModuleManager to load this module and its dependencies
        $config = [
            'module_listener_options' => [
                'module_paths' => $zf2ModulePaths,
            ],
            'modules' => [
                'Album',
            ]
        ];

        $serviceManager = new ServiceManager(new ServiceManagerConfig());
        $serviceManager->setService('ApplicationConfig', $config);
        $serviceManager->get('ModuleManager')->loadModules();
        static::$serviceManager = $serviceManager;
    }

    public static function chroot()
    {
        $rootPath = dirname(static::findParentPath('module'));
        chdir($rootPath);
    }

    public static function getServiceManager()
    {
        return static::$serviceManager;
    }

    protected static function initAutoloader()
    {
        $vendorPath = static::findParentPath('vendor');

        if (file_exists($vendorPath.'/autoload.php')) {
            include $vendorPath.'/autoload.php';
        }

        if (! class_exists('Zend\Mvc\Application')) {
            throw new RuntimeException('Unable to load ZF2. Run `composer install`');
        }
    }

    protected static function findParentPath($path)
    {
        $dir = __DIR__;
        $previousDir = '.';
        while (! is_dir($dir . '/' . $path)) {
            $dir = dirname($dir);
            if ($previousDir === $dir) {
                return false;
            }
            $previousDir = $dir;
        }
        return $dir . '/' . $path;
    }
}

Bootstrap::init();
Bootstrap::chroot();
```

The contents of this bootstrap file can be daunting at first sight, but all it
does is ensure that all the necessary files are autoloadable for our tests, and
that we have initial application configuration. The most important lines are in
the `init()` method, where we specify the `modules` for the application.  In
this case we are only loading the `Album` module as it has no dependencies
against other modules, but when testing other modules, you will need to factor
in their own dependencies.

Now, if you navigate to the `zf2-tutorial/module/Album/test/` directory, and run
`phpunit`, you should get a similar output to this:

```text
PHPUnit 5.3.4 by Sebastian Bergmann and contributors.

Configuration read from /var/www/zf2-tutorial/module/Album/test/phpunit.xml.dist

Time: 0 seconds, Memory: 1.75Mb

No tests executed!
```

Even though no tests were executed, we at least know that the autoloader found
the required files and classes; otherwise it would throw a `RuntimeException`,
as demonstrated in the `initAutoloader()` method.

## Your first controller test

Testing controllers is never an easy task, but the zend-test component makes
testing much less cumbersome.

First, create `AlbumControllerTest.php` under
`zf2-tutorial/module/Album/test/Controller` with the following contents:

```php
<?php
namespace AlbumTest\Controller;

use Zend\Test\PHPUnit\Controller\AbstractHttpControllerTestCase;

class AlbumControllerTest extends AbstractHttpControllerTestCase
{
    public function setUp()
    {
        $this->setApplicationConfig(
            // Grabbing the full application configuration:
            include __DIR__ . '/../../../../config/application.config.php'
        );
        parent::setUp();
    }
}
```

The `AbstractHttpControllerTestCase` class we extend here helps us setting up
the application itself, helps with dispatching and other tasks that happen
during a request, and offers methods for asserting request params, response
headers, redirects, and more. See the [zend-test](https://zendframework.github.io/zend-test/phpunit/)
documentation for more information.

The principal requirement for any zend-test test case is to set the application
config with the `setApplicationConfig()` method.

Now, add the following function to the `AlbumControllerTest` class:

```php
public function testIndexActionCanBeAccessed()
{
    $this->dispatch('/album');
    $this->assertResponseStatusCode(200);

    $this->assertModuleName('Album');
    $this->assertControllerName('Album\Controller\Album');
    $this->assertControllerClass('AlbumController');
    $this->assertMatchedRouteName('album');
}
```

This test case dispatches the `/album` URL, asserts that the response code is
200, and that we ended up in the desired module and controller.

> ### Assert against controller service names
>
> For asserting the *controller name* we are using the controller name we
> defined in our  routing configuration for the Album module. In our example
> this should be defined on line 19 of the `module.config.php` file in the Album
> module.

## A failing test case

Finally, `cd` to `zf2-tutorial/module/Album/test/` and run `phpunit`. Uh-oh! The
test failed!

```text
PHPUnit 5.3.4 by Sebastian Bergmann and contributors.

Configuration read from /var/www/zf2-tutorial/module/Album/test/phpunit.xml.dist

F

Time: 0 seconds, Memory: 8.50Mb

There was 1 failure:

1) AlbumTest\Controller\AlbumControllerTest::testIndexActionCanBeAccessed
Failed asserting response code "200", actual status code is "500"

/var/www/zf2-tutorial/vendor/ZF2/library/Zend/Test/PHPUnit/Controller/AbstractControllerTestCase.php:373
/var/www/zf2-tutorial/module/Album/test/AlbumTest/Controller/AlbumControllerTest.php:22

FAILURES!
Tests: 1, Assertions: 0, Failures: 1.
```

The failure message doesn't tell us much, apart from that the expected status
code is not 200, but 500. To get a bit more information when something goes
wrong in a test case, we set the protected `$traceError` member to `true`. Add
the following just above the `setUp` method in our `AlbumControllerTest` class:

```php
protected $traceError = true;
```

Running the `phpunit` command again and we should see some more information
about what went wrong in our test. The main error message we are interested in
should read something like:

```text
Zend\ServiceManager\Exception\ServiceNotFoundException: Zend\ServiceManager\ServiceManager::get
was unable to fetch or create an instance for Zend\Db\Adapter\Adapter
```

From this error message it is clear that not all our dependencies are available
in the service manager. Let us take a look how can we fix this.

## Configuring the service manager for the tests

The error says that the service manager can not create an instance of a database
adapter for us. The database adapter is indirectly used by our
`Album\Model\AlbumTable` to fetch the list of albums from the database.

The first thought would be to create an instance of an adapter, pass it to the
service manager, and let the code run from there as is. The problem with this
approach is that we would end up with our test cases actually doing queries
against the database. To keep our tests fast, and to reduce the number of
possible failure points in our tests, this should be avoided.

The second thought would be then to create a mock of the database adapter, and
prevent the actual database calls by mocking them out. This is a much better
approach, but creating the adapter mock is tedious (but no doubt we will have to
create it at some point).

The best thing to do would be to mock out our `Album\Model\AlbumTable` class
which retrieves the list of albums from the database. Remember, we are now
testing our controller, so we can mock out the actual call to `fetchAll` and
replace the return values with dummy values. At this point, we are not
interested in how `fetchAll` retrieves the albums, but only that it gets called
and that it returns an array of albums; these facts allow us to provide mock
instances. When we test `AlbumTable` itself, we can write the actual tests for
the `fetchAll` method.

Here is how we can accomplish this, by modifying the
`testIndexActionCanBeAccessed` test method as follows:

```php
public function testIndexActionCanBeAccessed()
{
    $albumTableMock = $this->getMockBuilder(AlbumTable::class)
        ->disableOriginalConstructor()
        ->getMock();

    $albumTableMock->expects($this->once())
        ->method('fetchAll')
        ->will($this->returnValue([]));

    $serviceManager = $this->getApplicationServiceLocator();
    $serviceManager->setAllowOverride(true);
    $serviceManager->setService(AlbumTable::class, $albumTableMock);

    $this->dispatch('/album');
    $this->assertResponseStatusCode(200);

    $this->assertModuleName('Album');
    $this->assertControllerName('Album\Controller\Album');
    $this->assertControllerClass('AlbumController');
    $this->assertMatchedRouteName('album');
}
```

By default, the `ServiceManager` does not allow us to replace existing services.
As the `Album\Model\AlbumTable` was already set, we are allowing for overrides
(via the `setAllowOverride()` call), and then replacing the real instance of the
`AlbumTable` with a mock.  The mock is created so that it will return just an
empty array when the `fetchAll` method is called. This allows us to test for
what we care about in this test, and that is that by dispatching to the `/album`
URL we get to the Album module's AlbumController.

Running `phpunit` at this point, we will get the following output as the tests
now pass:

```text
PHPUnit 5.3.4 by Sebastian Bergmann and contributors.

Configuration read from /var/www/zf2-tutorial/module/Album/test/phpunit.xml.dist

.

Time: 0 seconds, Memory: 9.00Mb

OK (1 test, 6 assertions)
```

## Testing actions with POST

One of the most common actions happening in controllers is submitting a form
with some POST data; those tests look similar to the following:

```php
 :linenos:}
public function testAddActionRedirectsAfterValidPost()
{
    $albumTableMock = $this->getMockBuilder(AlbumTable::class)
        ->disableOriginalConstructor()
        ->getMock();

    $albumTableMock->expects($this->once())
        ->method('saveAlbum')
        ->will($this->returnValue(null));

    $serviceManager = $this->getApplicationServiceLocator();
    $serviceManager->setAllowOverride(true);
    $serviceManager->setService(AlbumTable::class, $albumTableMock);

    $postData = [
        'title'  => 'Led Zeppelin III',
        'artist' => 'Led Zeppelin',
        'id'     => '',
    ];
    $this->dispatch('/album/add', 'POST', $postData);
    $this->assertResponseStatusCode(302);

    $this->assertRedirectTo('/album/');
}
```

Here we test that when we make a POST request against the `/album/add` URL, the
`Album\Model\AlbumTable`'s `saveAlbum()` method will be called, and after that
we will be redirected back to the `/album` URL.

Running `phpunit` gives us the following output:

```text
PHPUnit 5.3.4 by Sebastian Bergmann and contributors.

Configuration read from /home/robert/www/zf2-tutorial/module/Album/test/phpunit.xml.dist

..

Time: 0 seconds, Memory: 10.75Mb

OK (2 tests, 9 assertions)
```

Testing the `editAction()` and `deleteAction()` methods can be easily done in a
manner similar as shown for the `addAction()`.

When testing the `editAction()` method, you will also need to mock out the
`getAlbum()` method:

```php
$albumTableMock->expects($this->once())
    ->method('getAlbum')
    ->will($this->returnValue(new Album()));
```

## Testing model entities

Now that we know how to test our controllers, let us move to an other important
part of our application: the model entity.

Here we want to test that the initial state of the entity is what we expect it
to be, that we can convert the model's parameters to and from an array, and that
it has all the input filters we need.

Create the file `AlbumTest.php` in `module/Album/test/Model` directory
with the following contents:

```php
<?php
namespace AlbumTest\Model;

use Album\Model\Album;
use PHPUnit_Framework_TestCase as TestCase;

class AlbumTest extends TestCase
{
    public function testAlbumInitialState()
    {
        $album = new Album();

        $this->assertNull(
            $album->artist,
            '"artist" should initially be null'
        );

        $this->assertNull(
            $album->id,
            '"id" should initially be null'
        );

        $this->assertNull(
            $album->title,
            '"title" should initially be null'
        );
    }

    public function testExchangeArraySetsPropertiesCorrectly()
    {
        $album = new Album();
        $data  = [
            'artist' => 'some artist',
            'id'     => 123,
            'title'  => 'some title'
        ];

        $album->exchangeArray($data);

        $this->assertSame(
            $data['artist'],
            $album->artist,
            '"artist" was not set correctly'
        );

        $this->assertSame(
            $data['id'],
            $album->id,
            '"id" was not set correctly'
        );

        $this->assertSame(
            $data['title'],
            $album->title,
            '"title" was not set correctly'
        );
    }

    public function testExchangeArraySetsPropertiesToNullIfKeysAreNotPresent()
    {
        $album = new Album();

        $album->exchangeArray([
            'artist' => 'some artist',
            'id'     => 123,
            'title'  => 'some title',
        ]);
        $album->exchangeArray([]);

        $this->assertNull(
            $album->artist,
            '"artist" should have defaulted to null'
        );

        $this->assertNull(
            $album->id,
            '"id" should have defaulted to null'
        );

        $this->assertNull(
            $album->title,
            '"title" should have defaulted to null'
        );
    }

    public function testGetArrayCopyReturnsAnArrayWithPropertyValues()
    {
        $album = new Album();
        $data  = [
            'artist' => 'some artist',
            'id'     => 123,
            'title'  => 'some title'
        ];

        $album->exchangeArray($data);
        $copyArray = $album->getArrayCopy();

        $this->assertSame(
            $data['artist'],
            $copyArray['artist'],
            '"artist" was not set correctly'
        );

        $this->assertSame(
            $data['id'],
            $copyArray['id'],
            '"id" was not set correctly'
        );

        $this->assertSame(
            $data['title'],
            $copyArray['title'],
            '"title" was not set correctly'
        );
    }

    public function testInputFiltersAreSetCorrectly()
    {
        $album = new Album();

        $inputFilter = $album->getInputFilter();

        $this->assertSame(3, $inputFilter->count());
        $this->assertTrue($inputFilter->has('artist'));
        $this->assertTrue($inputFilter->has('id'));
        $this->assertTrue($inputFilter->has('title'));
    }
}
```

We are testing for 5 things:

1. Are all of the `Album`'s properties initially set to `NULL`?
2. Will the `Album`'s properties be set correctly when we call `exchangeArray()`?
3. Will a default value of `NULL` be used for properties whose keys are not present in the `$data` array?
4. Can we get an array copy of our model?
5. Do all elements have input filters present?

If we run `phpunit` again, we will get the following output, confirming that our
model is indeed correct:

```text
PHPUnit 5.3.4 by Sebastian Bergmann and contributors.

Configuration read from /var/www/zf2-tutorial/module/Album/test/phpunit.xml.dist

.......

Time: 0 seconds, Memory: 11.00Mb

OK (7 tests, 25 assertions)
```

## Testing model tables

The final step in this unit testing tutorial for zend-mvc applications is
writing tests for our model tables.

This test assures that we can get a list of albums, or one album by its ID, and
that we can save and delete albums from the database.

To avoid actual interaction with the database itself, we will replace certain
parts with mocks.

Create a file `AlbumTableTest.php` in `module/Album/test/Model` with
the following contents:

```php
<?php
namespace AlbumTest\Model;

use Album\Model\AlbumTable;
use Album\Model\Album;
use PHPUnit_Framework_TestCase as TestCase;
use Zend\Db\ResultSet\ResultSet;

class AlbumTableTest extends TestCase
{
    public function testFetchAllReturnsAllAlbums()
    {
        $resultSet = new ResultSet();
        $mockTableGateway = $this->getMock(
            'Zend\Db\TableGateway\TableGateway',
            ['select'],
            [],
            '',
            false
        );
        $mockTableGateway->expects($this->once())
            ->method('select')
            ->with()
            ->will($this->returnValue($resultSet));

        $albumTable = new AlbumTable($mockTableGateway);

        $this->assertSame($resultSet, $albumTable->fetchAll());
    }
}
```

Since we are testing the `AlbumTable` here and not the `TableGateway` class
(which has already been tested in zend-db), we only want to make sure
that our `AlbumTable` class is interacting with the `TableGateway` class the way
that we expect it to. Above, we're testing to see if the `fetchAll()` method of
`AlbumTable` will call the `select()` method of the `$tableGateway` property
with no parameters. If it does, it should return a `ResultSet` object. Finally,
we expect that this same `ResultSet` object will be returned to the calling
method. This test should run fine, so now we can add the rest of the test
methods:

```php
public function testCanRetrieveAnAlbumByItsId()
{
    $album = new Album();
    $album->exchangeArray([
        'id'     => 123,
        artist' => 'The Military Wives',
        title'  => 'In My Dreams'
    ]);

    $resultSet = new ResultSet();
    $resultSet->setArrayObjectPrototype(new Album());
    $resultSet->initialize(array($album));

    $mockTableGateway = $this->getMock(
        'Zend\Db\TableGateway\TableGateway',
        ['select'],
        [],
        '',
        false
    );
    $mockTableGateway->expects($this->once())
        ->method('select')
        ->with(['id' => 123])
        ->will($this->returnValue($resultSet));

    $albumTable = new AlbumTable($mockTableGateway);

    $this->assertSame($album, $albumTable->getAlbum(123));
}

public function testCanDeleteAnAlbumByItsId()
{
    $mockTableGateway = $this->getMock(
        'Zend\Db\TableGateway\TableGateway',
        ['delete'],
        [],
        '',
        false
    );
    $mockTableGateway->expects($this->once())
        ->method('delete')
        ->with(['id' => 123]);

    $albumTable = new AlbumTable($mockTableGateway);
    $albumTable->deleteAlbum(123);
}

public function testSaveAlbumWillInsertNewAlbumsIfTheyDontAlreadyHaveAnId()
{
    $albumData = [
        'artist' => 'The Military Wives',
        'title'  => 'In My Dreams'
    ];
    $album = new Album();
    $album->exchangeArray($albumData);

    $mockTableGateway = $this->getMock(
        'Zend\Db\TableGateway\TableGateway',
        ['insert'],
        [],
        '',
        false
    );
    $mockTableGateway->expects($this->once())
        ->method('insert')
        ->with($albumData);

    $albumTable = new AlbumTable($mockTableGateway);
    $albumTable->saveAlbum($album);
}

public function testSaveAlbumWillUpdateExistingAlbumsIfTheyAlreadyHaveAnId()
{
    $albumData = [
        'id'     => 123,
        'artist' => 'The Military Wives',
        'title'  => 'In My Dreams',
    ];
    $album = new Album();
    $album->exchangeArray($albumData);

    $resultSet = new ResultSet();
    $resultSet->setArrayObjectPrototype(new Album());
    $resultSet->initialize([$album]);

    $mockTableGateway = $this->getMock(
        'Zend\Db\TableGateway\TableGateway',
        ['select', 'update'],
        [],
        '',
        false
    );
    $mockTableGateway->expects($this->once())
        ->method('select')
        ->with(['id' => 123])
        ->will($this->returnValue($resultSet));
    $mockTableGateway->expects($this->once())
        ->method('update')
        ->with(
            [
                'artist' => 'The Military Wives',
                'title'  => 'In My Dreams'
            ],
            ['id' => 123]
        );

    $albumTable = new AlbumTable($mockTableGateway);
    $albumTable->saveAlbum($album);
}

public function testExceptionIsThrownWhenGettingNonExistentAlbum()
{
    $resultSet = new ResultSet();
    $resultSet->setArrayObjectPrototype(new Album());
    $resultSet->initialize([]);

    $mockTableGateway = $this->getMock(
        'Zend\Db\TableGateway\TableGateway',
        ['select'],
        [],
        '',
        false
    );
    $mockTableGateway->expects($this->once())
        ->method('select')
        ->with(['id' => 123])
        ->will($this->returnValue($resultSet));

    $albumTable = new AlbumTable($mockTableGateway);

    $this->setExpectedException('Exception', 'Could not find row 123');
    $albumTable->getAlbum(123);
}
```

These tests are nothing complicated and they should be self explanatory. In each
test we are injecting a mock table gateway into our `AlbumTable`, and we then
set our expectations accordingly.

We are testing that:

1. We can retrieve an individual album by its ID.
2. We can delete albums.
3. We can save a new album.
4. We can update existing albums.
5. We will encounter an exception if we're trying to retrieve an album that doesn't exist.

Running `phpunit` one last time, we get the output as follows:

```text
PHPUnit 5.3.4 by Sebastian Bergmann and contributors.

Configuration read from /var/www/zf2-tutorial/module/Album/test/phpunit.xml.dist

.............

Time: 0 seconds, Memory: 11.50Mb

OK (13 tests, 34 assertions)
```

## Conclusion

In this short tutorial, we gave a few examples how different parts of a zend-mvc
application can be tested. We covered setting up the environment for testing,
how to test controllers and actions, how to approach failing test cases , how to
configure the service manager, as well as how to test model entities and model
tables .

This tutorial is by no means a definitive guide to writing unit tests, just a
small stepping stone helping you develop applications of higher quality.
