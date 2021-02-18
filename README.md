# Architecture

## General organization

### Technical domain first, business domain then

Organize the folders by thinking **technical domain first**.

E.g. bad organization:

    src/
    ├─ Website1/
    │  ├─ Fetcher.php
    │  └─ Parser.php
    └─ Website2/
       ├─ Fetcher.php
       └─ Parser.php
E.g. good organization:

    src/
    ├─ Fetcher/
    │  ├─ Website1Fetcher.php
    │  └─ Website2Fetcher.php
    └─ Parser/
       ├─ Website1Parser.php
       └─ Website2Parser.php
 
## Controllers

### Thin controllers, multiple thin services

Well, the common rule is *"Thin controllers, fat services"*. However, "fat services" sounds like an **unmaintainable** pattern. Better create a **lot of thin services** with a **very few main functions**.

Controllers, as other entry points patterns such  Commands or EventSubscribers **should not contain any business logic**.  They should only handle **services**.

**Advantages**: maintainability, reusability, separation of concerns

## Services

### One service, one function

As far as possible, services should expose only **one main public method**.

E.g.:
- The aim of an `ObjectHandler` is to `handle()`
- The aim of an `ObjectFetcher` is to `fetch()`
- The aim of an `ObjectParser` is to `parse()`

**Advantages**: maintainability, understanding

### Services are stateless

Services should **not persist any business data**. Recipes depending on setting data before handling it is an anti-pattern: what a mental load... Only some **configuration** and **other services** should be injected and set as attributes of a service.

E.g. **bad** pattern:

    public class UserService
    {
        private User $user
        
        public function handleUser(): void
        {
            $this->user->method();
        }

        public function setUser(User $user): void
        {
            $this->user = $user
        }
    }

E.g. **good** pattern:

    public class UserService
    {   
        public function handleUser(User $user): void
        {
            $user->method();
        }
    }
    
**Advantages**: stateless, no mental load

## API clients

All logic making HTTP **requests to a given API** should be **located in a specific class** (an API client, or API wrapper)
Each **endpoint call** should be **wrapped into a specific function**.

E.g.:

    class GoogleMapsClient
    {
        private const ENDPOINT_GET_PLACE = '/path/to/place/endpoint';

        private $client;	// HTTP client
        
        public function getPlace(string $placeId): array
        {
            try {
                $response = $this->client->get(self::ENDPOINT_GET_PLACE . $placeId);
            } catch (RequestException $e) {
                throw new GoogleMapsException($e->getMessage);	// Throw a custom exception
            }
            
            return json_decode($response->getBody()->getContents(), true);
        }
    }

**Advantages**: reusability, separation of concerns, easily mockable in UTs

## Abstract classes vs Interfaces

**Abstract classes are not interfaces**. Both provide a way to define **contracts** in a polymorphism context. However, their scope is different.

Abstract classes are sometimes used as interfaces because they **assert** that the given polymorphic object **implements** some needed abstract methods.

But they should not, as abstract methods are designed to achieve **internal contracts** between the Abstract class and its extended classes. Only Interfaces should be used as contracts in an **external context.** BTW, make an Abstract class implement an Interface is absolutely a good practice (and not redundant).

E.g.: ~~`public function __construct(AbstractService $service)`~~ ---> `public function __construct(ServiceInterface $service)`

**Advantages**: pattern respect

## DTO vs DAO

In a Doctrine context, an *Entity* is a DAO:  it represents a **persisted** object **mapped** to a database table.
Sometimes we handle unpersisted data, for temporary needs. We could store them in arrays. However, arrays are **not very structured** and can't **respect any contract**.

Working with **objects** is often the best way to achieve it. That's what DTO are designed for: they are very simple objects containing business **structured data** for **temporary needs** (their life cycle is short). They should be ***read-only*** as far as possible, as they just exist to be consumed.

Complex DTOs can be created using a *Factory*.

## Utils

As known as "Helpers". **Low-level functions** handling primary types ( `string`, `array`, etc..) or simple native objects such `DateTime` should be located in a dedicated namespace such `Utils` (like [Symfony demo](https://github.com/symfony/demo/tree/master/src/Utils)) and in dedicated classes.

Pretty often this kind of low-level technical logic is totally **uncorrelated** from the main business logic. Because these classes are totally **decoupled** and don't need any dependency to be used, its methods should be declared `static` so that they can be used everywhere, whenever, without instantiating anything.

E.g.: `StringUtils::slugify()`, `LocationUtils::getDistance()`, `NumericUtils::extractFloatFromString()`
	
**Advantages**: reusability, separation of concerns

# Data

## Database and application

**A database should be decoupled from any application**. It's a stand-alone part of an information system. In other words, it should be plugged in and consumed by any application. The data structure should **not be driven by the application**.

**Advantages**: Better decoupling

## Static data

**No static data in business database**. Static data are fixed data: once defined, they're **rarely updated.** They are defined by developers themselves, so they are part of the **application configuration** and should not be stored in a business-oriented database.

**No static data in classes.** In order to **separate clearly source code and data**, structured static data should not be stored as constants or static variables in classes or interfaces.

Static data should be defined in configuration files (eg : YAML files) or in a embedded SQLite database. However, it's OK to reference them in a business database, like a foreign key. Because a database should be readable and understandable by a human all the time, using integer IDs for the reference type does not seem a good solution. Better use a string identifier such a **slug**.

**Advantages**: Separation of concerns

## Calculated data

**No calculated fields in database**. Calculated fields which **entirely depend** on an other field or an other table should not be stored in database. They **look like useful shortcuts** when doing a SQL request... Actually, they're just  a **mental load** and a **bug source** in the application.

E.g.: 

    +-------------------------------------------------+
    |                     comment                     |
    +----------+-----------+--------------+-----------+
    | reply_id | has_reply | published_at | published |
    +----------+-----------+--------------+-----------+

Information `has_reply`  and  `is_published`  can be **calculated** from respectively `reply_id` and `published_at` so they should not exist in the table.

It would be a mental load to set them if they're used in your application: "*This comment has a new reply, I don't have to forget to set `has_reply = 1` (even if `reply_id` is already set). The comment has been published, I don't have to forget to set `published = 1` (even if `published_at` is already set)*"

The solution is **application-related only**, not database-related:  create getters/isers/hasers in the Doctrine entity without defining persisted attributes:

    class Comment {
       private ?Reply $reply;
       private ?DateTime $publishedAt;
    
       public function hasReply(): bool
       {
           return null !== $this->reply;
       }
       
       public function isPublished(): bool
       {
           return null !== $this->publishedAt;
       }
    }
 
**Advantages**: No potential bugs, no mental load, lighter database

## Arrays: collection vs hash table

An array can represent mainly two data structures: a collection (of objects or hash tables) and a hash table (object as an array).  
 
- A collection can be empty (`[]`) but **should never be `null`**: it should be **iterable all the time**, without checking for *not null* first
- A hash table can be `null` but **should never be empty (`[]`)**: it exists or it does not

# Code

## Conditions

### Ternary operator

### Checking for null

When testing that a variable is `null` or not `null`,  **don't test the type of the variable**. Be direct, be explicit.
E.g.: ~~`if (!is_numeric($number))`~~ ---> `if (null !== $number)`

**Advantages**: No mental load. "*Did the developer mean that this variable could have multiple types? Or did he just want to check for `null`/`!null`?*" 


## Exceptions

 ### Bubble up the exceptions

Let the exceptions **bubble up** (raise) as far as possible and catch them at the **highest level** (entry points) (eg: Controller, Command, Consumer...) Do not catch them at a low level in order to return `null` , `false`,  `[]` , etc... : the information that **something went wrong** would be **lost**.

**Advantages**: Maintainability: no more wondering: "*Does the function return `null` because it should really return `null` or because an error occurred?*". Flexibility: process separately different kinds of exception in highest levels.

### Loop case

If an instruction throw an exception inside a loop structure (eg:  `foreach` or `for`) you probably don't want to **break the execution** at this point, even in a service. It's often a better solution to catch the exception and go to the **next iteration**.

E.g.:

    foreach ($data as $item) {
        try {
            function($item);
        } catch (Exception $e) {
            // Do something
            
            continue;
        }
    }

### Try/catch structure



## Private vs protected

**Protected is not private-like**. `private` **first**, `protected` **if needed**. Each visibility has its purpose. `protected` should be used only when attributes/methods are intended to be used/overriden in extended classes

## Over-testing

## Nullability and context 

Sometimes a **nullable** variable/function (declared as it is in the PHPDoc or through the type hinting) is definitively **not null in a given context**. However, IDEs and Continuous Integration tools can not know that and ask us to check for nullability.

Adding a test is not a very good solution: this is a useless condition (so useless code lines) because we know we'll **never enter inside** the `if` block. It looks like **hypocrisy**. Furthermore, it gives a **false information**: "*Really? It could be null here?*" No, it could not.

It's better to force "not null"  through the PHPDoc:

    /** @var Object $var **/
    $var = functionWhichCanReturnNullButNotIsThisContextTrustMe();

**Advantages**: No useless condition,  so no new complexity

## Avoid discontinued threads 

# Naming conventions

## Constants

**Group constants by set** as far as possible. Name the constants like a **namespace**: prefix them with the qualified concept, finish with the value.

E.g.: 

    public const STATUS_PENDING = 'pending';
    public const STATUS_STARTED = 'started';
    public const STATUS_FINISHED = 'finished';

**Advantages**: readability, understanding

## Service's methods

If a service contains the qualified concept in its name, **do not repeat** this qualified concept in its methods.
The service name is self-sufficient. Furthermore, it's "interface-friendly".

E.g. : ~~`FileFinder::findFile()`~~ ---> `FileFinder::find()` 

**Advantages**: no redundancy, Interface-oriented

## Twig

- Use **snake_case** for templates names, directories, and variables (eg: `/real_property/edit_price.html.twig`)
- Prefix template fragments (also called partial templates) with an **underscore** (eg: `_notification.html.twig`)

**Advantages**: Better separation between complete page templates and reusable components,   [Symfony best practices](https://symfony.com/doc/current/best_practices.html#templates)

## Others

- **Array**:  Suffix collections with a `s` (eg: `$comments` for an array containing `Comment` objects)
- **DateTime**: Suffix DateTime objects with `At` (eg: `$publishedAt`) (Symfony style)
- **Boolean**: While the context is convenient, suffix the variables using the simple past tense (eg: `$sent`, `$published`). Don't prefix class attributes with `$is`, do it only for the "isers" (eg: `isSent()`, `isPublished()`)
- **Counter**: ~~`$numberOfItems`~~ is very verbose. Use `$itemCount` instead

# Code style

## Object instanciation

Apply fluent setters on the `__construct()` return to build an object in an elegant way

    $object = (new Object)
        ->setAttribute1($val1)
        ->setAttribute2($val2)
        ->setAttribute3($val3);
        
**Advantages:** code lines saving

##  "if" structure

Finish all conditions with `&&` or `||` operators

    if (
        longCondition1 &&
        longCondition2 ||
        (
            longCondition3 &&
            longCondition4
        )
    ) {
        // Do something
    }
    
**Advantages**: readability	

## Instruction with mandatory arguments and optional parameters

Like Symfony `render()` method, put the mandatory arguments on **one line** and the parameters array on **several lines**:

    $object->method($arg1, $arg2, $arg3, [
       $param1 => XXX,
       $param2 => XXX,
       $param3 => XXX
    ]);

**Advantages**: Symfony style, readability

## Others

- **return**: New line before final instructions (`return`, `throw`, `continue`, `break`, `exit`) (**Advantages**: PSR, readability)
- `'` instead of `"`
- `$array = []` instead of `$array = array()`
- Twig `{{ include() }}` function instead of `{% include %}` tag (**Advantages**: more flexibility)


