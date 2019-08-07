# PHP Code refactor for unit-testing


## Dependency injection

The use of dependency injection does not only allow you to write better tests, it will also make your codebase more maintainable over the time.

Let’s look at quick (bad) code example. Imagine we are developing a software that manages table reservations for restaurants. For sake of simplicity, we will show only constructors of the Client and Reservation classes.

Code Example 1
```
class Client
{
    private $firstName; 
    private $lastName;
    
    public function __construct($firstName, $lastName)
    {
        $this->firstName = $firstName; 
        $this->lastName = $lastName;
    }
    
    // other method declarations would go here
}

class Reservation
{
    private $numberOfPersons; 
    private $client;
    
    public function __construct($numberOfPersons, $firstName, $lastName)
    {
        $this->client = new Client($firstName, $lastName);
        $this->numberOfPersons = $numberOfPersons;
    }
    
    // other method declarations would go here
}
```

This short code example is enough to demonstrate the problem: 
If you want to test the Reservation class, you will have to test the Client class and, with this, your Reservation tests may fail due to errors on the Client class.

This will also hurt code maintainability. Client is tightly coupled with the Reservation. So, if you change the Client constructor signature you would also need to change the Reservation class.

So, a better example of the Reservation constructor would be:

Code Example 2
```
public function __construct($numberOfPersons, Client $client)
{
     $this->client = $client;
     $this->numberOfPersons = $numberOfPersons;
}
```

This way, in your tests, you can create a mock object of the Client class and you will be testing only the Reservation class as it should be!

This same principle should be applied to global variables. If you use global variables inside any of your classes you will be coupling variables on your application and you will be running into problems during testing because you will have external dependencies that you will also need to test and guarantee they are present.

The global keyword should not be used, and once again you should inject any dependencies from the outside of your methods and not the other way around.

So, wrapping it up, avoid coupling as much as you can, this will make your code more testable, you will use mock objects, and will also save you a lot of trouble in future when you need to refactor (and you know you will need to do it).


## Each function has one function

    Functions should do one thing. They should do it well. They should do it only.

If you respect Single Responsibility Principle, you will know exactly what you are testing. Your functions will be smaller and your codebase simpler.

Here’s some of the problems you face when a function does a lot of stuff:

    The function can have side effects you wouldn’t assume by it’s name
    The function code get’s too complex and hard to maintain over time
    There’s an high probability that some of the functionality there would be repeated somewhere in the code because you don’t have a separate method to do it (see DRY principle)
    And, more important for the context of this article, it will be more difficult to write unit tests for it because it will be harder to evaluate what happens inside that function

So let’s look at another bad code example of a method on the Reservation class:

Example code 3
```
public function saveReservation() 
{
    $this->idClient = $this->client->save();
    $this->save();
    $this->email->sendEmail();
}
```

This function clearly does more than one thing. When you call the method saveReservation, it is also saving the Client and sending an email.

So, in fact, when you test this method you will be testing more stuff than the saving of the reservation. You will also be testing saving a Client and sending an email. Even if you correctly injected all the dependencies necessary for these actions, the effort to write tests is much bigger because of the increased complexity of the method.
Functions should have a minimum number of arguments

The more arguments your methods have, the more complex will be the test because you will have to test all the possible combinations of things that might change in with each one of that arguments combinations.

Code example 4
```
class Reservation
{
    private $numberOfPersons; 
    private $client;
    private $db; 
    
    public function __construct(Database $db)
    {
        global $db;
    }
    
    public function setClient(Client $client)
    {
        $this->client = $client;
    }
    
    public function setNumberOfPersons($numberOfPersons)
    {
        $this->numberOfPersons = $numberOfPersons;
    }
    
    // other method declarations would go here
}
```

## Functions depending on the global state

Global variables are a convenience in PHP applications. 
They allow you to have variables or objects that can be initialized early in your application, and then can be leveraged anywhere in the application. However, that flexibility comes at a cost, as the heavy use of global variables is a common problem seen in untestable code. We can see this in Listing 1.

Code example 4. Function that depends upon the global state
```
<?php 
function formatNumber($number) 
{ 
    global $decimal_precision, $decimal_separator, $thousands_separator; 
      
    if ( !isset($decimal_precision) ) $decimal_precision = 2; 
    if ( !isset($decimal_separator) ) $decimal_separator = '.'; 
    if ( !isset($thousands_separator) ) $thousands_separator = ','; 
      
    return number_format($number, $decimal_precision, $decimal_separator, 
$thousands_separator); 
}
```

Two different issues arise because of these global variables. 
The first issue is that you need to account for each of them in your test, making sure you set them to valid values that the function expects. The second and bigger issue is so that you don't change the state on subsequent tests and invalidate their results you need to make sure that you reset the global state back to how it was before the test was run. PHPUnit has facilities that can backup global variables and restore them after your test is run, which can help alleviate the issue. However, a better approach is to provide a way for the tester class to directly pass in values for these globals that the method can use. Listing 2 shows an example of how to do this.

Code example 5. Fixed this function to allow overriding the global variables
```
<?php 
function formatNumber($number, $decimal_precision = null, $decimal_separator = null, 
$thousands_separator = null) 
{ 
    if ( is_null($decimal_precision) ) global $decimal_precision; 
    if ( is_null($decimal_separator) ) global $decimal_separator; 
    if ( is_null($thousands_separator) ) global $thousands_separator; 
      
    if ( !isset($decimal_precision) ) $decimal_precision = 2; 
    if ( !isset($decimal_separator) ) $decimal_separator = '.'; 
    if ( !isset($thousands_separator) ) $thousands_separator = ','; 
      
    return number_format($number, $decimal_precision, $decimal_separator, 
$thousands_separator);
}
```

Doing this not only has made the code more testable, 
but also made it not depend on the global variables in the method. 
This can open up the possibility of refactoring this code down the road to not use the global variables at all.


## Singletons that can't be reset

Singletons are classes that are designed to have only one instance existing at a time in an application. 
They are a common pattern used for global objects in an application, such as database connections and configuration settings. 
They are often considered taboo in an application, which many developers consider to be an undeserving distinction, 
because of the usefulness of having an always available object for use. 
Much of this comes from the overuse of singletons, where many of these so called god objects can be impossible to extend from. 
But from a testing perspective, a big problem is that they are often immutable. 
Let's look at Code example 6 as an example.

Code example 6. Singleton object we are looking to test
```
<?php 
class Singleton 
{ 
    private static $instance; 
      
    protected function __construct() { } 
    private final function __clone() {} 
      
      
    public static function getInstance() 
    { 
        if ( !isset(self::$instance) ) { 
            self::$instance = new Singleton; 
        } 
          
        return self::$instance; 
    } 
}
```

So you can see that after the singleton is instantiated the first time, 
every call made to the getInstance() method will return back the same object and not a new one, 
which can become a big problem if we make changes to that object. 
The easiest solution is to add a method to the object that can reset it. 
Code example 7 shows such an example.

Code example 7. Singleton object with a reset method added
```	
<?php 
class Singleton 
{ 
    private static $instance; 
      
    protected function __construct() { } 
    private final function __clone() {} 
      
      
    public static function getInstance() 
    { 
        if ( !isset(self::$instance) ) { 
            self::$instance = new Singleton; 
        } 
          
        return self::$instance; 
    } 
      
    public static function reset() 
    { 
        self::$instance = null; 
    } 
}
```

Now, we can call the reset method to start off each test run to ensure we are going through the initialization code for the singleton object on every test run. 
Having this method available can be helpful in the application in general because now the singleton becomes easily mutable.
Working in the class constructor

A good practice when unit testing is to only test exactly what you are intending to, and avoid having to setup more objects and variables than you absolutely need. Every object and variable you set is also one you need to remove after the fact. This becomes a problem for more pesky items such as files and database tables, where if you need to modify the state you must be extra careful to cleanup your tracks after the test is completed. The biggest barrier to keeping that rule intact is the constructor of the object itself, which does all sorts of things that aren't pertinent to your test. 
Consider Code example 8 below for an example.

Code example 8. Class with a large singleton method
```
<?php
class MyClass 
{ 
    protected $results; 
      
    public function __construct() 
    { 
        $dbconn = new DatabaseConnection('localhost','user','password'); 
        $this->results = $dbconn->query('select name from mytable'); 
    } 
      
    public function getFirstResult() 
    { 
        return $this->results[0]; 
    } 
}
```

Here, in order to test the getFirstResult method in the object, we end up needing to setup a database connection, 
have records in the table, and then clean up all these resources after the fact. 
This seems like overkill when none of this is needed to test the fdfdfd method. 
Therefore, let's modify the constructor as shown in Listing Code example 8.

Code example 9. Class modified to optionally skip all the unneeded initialization logic
```
<?php
class MyClass 
{ 
    protected $results; 
      
    public function __construct($init = true) 
    { 
        if ( $init ) $this->init(); 
    } 
      
    public function init() 
    { 
        $dbconn = new DatabaseConnection('localhost','user','password'); 
        $this->results = $dbconn->query('select name from mytable');
    } 
      
    public function getFirstResult() 
    { 
        return $this->results[0]; 
    } 
}
```

We've refactored the large amount of code in the constructor to put it in an init() method, which still will be called by default in the constructor to avoid breaking any existing code. 
However, now we can just pass a boolean false to the constructor during our tests to avoid calling the init() method and all the unneeded initialization logic. 
This refactoring of the class also improves the code so that we have separated the initialization code from the object construction code.
Having hard coded class dependencies

As we saw in the previous section, a huge problem in class design that makes testing difficult is having to initialize all sorts of objects that aren't required for your test. 
Previously, we saw how heavy initialization logic can add all sorts of overhead into writing a test (especially when the test doesn't need any of this to succeed), 
but another problem can occur when we directly create new objects inside methods of the class we may be testing. Let's look at Code example 10 for an example of such problematic code.

Code example 10. Class that has a method that directly initializes another object
```
<?php
class MyUserClass 
{ 
    public function getUserList() 
    { 
        $dbconn = new DatabaseConnection('localhost','user','password'); 
        $results = $dbconn->query('select name from user'); 
          
        sort($results); 
          
        return $results; 
    } 
}
```

Let's say we are testing the getUserList method above, 
but the focus of our test is to make sure that the returned user list is properly sorted alphabetically. 
In this case the fact that we could grab the records from the database really doesn't matter, since what we would be testing is our ability to sort the records coming back. 
The problem is that since we directly instantiate a database connection object inside the method, we are required to do all this scaffolding work to properly test the method. 
Therefore, let's make a change to allow for an object to be interjected, as shown in Code example 11.

Code example 11. Class that has a method that directly initializes another object, but also provides a way to override it
```	
<?php
class MyUserClass 
{ 
    public function getUserList($dbconn = null) 
    { 
        if ( !isset($dbconn) || !( $dbconn instanceOf DatabaseConnection ) ) { 
            $dbconn = new DatabaseConnection('localhost','user','password'); 
        } 
        $results = $dbconn->query('select name from user'); 
          
        sort($results); 
          
        return $results; 
    } 
}
```

Now you can pass in an object directly that is compatible with the expected database connection object, and it will use that object instead of creating a new one. The object you passed can simply be a mock object, meaning one where we hard code several of the return values of called methods to directly provide back the data we wish to use. In this case, we would mock the query method of the database connection object so that we could just return back the results instead of calling out to the database for them. Doing this kind of refactoring can also can improve the method, allowing your application to interject different database connections instead of being tied only to the default one specified.
