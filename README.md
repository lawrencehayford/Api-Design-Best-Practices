# API DESIGN BEST PRATICES

## 1. Be stingy with data

``` 
$user = User::model()->findAll(array(
            'condition' => "username = 'larry'",
        ));
        
return $user;

```
Response

```
{
  "id": 10,
  "name": "Lawrence Casely-Hayford",
  "username": "larry",
  "friend_count": 1337,
  "avatar": "https://example.org/tlhunter.jpg",
  "updated": "2018-12-24T21:13:22.933Z",
  "hometown": "Ann Arbor, MI"
}

```

### Ways to avoid delivering every property of the table object and exposing db Schema

### I. write a function inside your User Model to return sepcific properties of the User Object

example
```
class User extends CActiveRecord
{
    /**
     * @access public
     * @param string $className
     * @return \CActiveRecord
     */
    public static function model($className = __CLASS__)
    {
        return parent::model($className);
    }

    /**
     * @access public
     * @return string
     */
    public function tableName()
    {
        return 'user';
    }

    /**
    * @access public
    * @return array
    */
    public function getUserDetails()
    {
          return json_encode (array(
            'id' => $this->id,
            'name' => $this->name,
            'friend_count' => $this->friend_count,
            'avatar' => $this->avatar,
        ));
    }
```

### II. When returing response, Strictly send the requested user properties

example
```
$user = User::model()->findAll(array(
            'condition' => "username = 'larry'",
        ));
        
return json_encode (array(
            'id' => $user->id,
            'name' => $user->name,
            'friend_count' => $user->friend_count,
            'avatar' => $user->avatar,
        ));

```

### III. One should always look at the business requirements and determine what the absolute minimum amount of data can be provided that satisfies those requirements.
What does the consumer of the API really need?
eg
```
SELECT * FROM users where username ='larry';
```

should rather be
```
SELECT * id, name, friend_count, avatar FROM users where username ='larry';
```


## 1. Represent upstream data as well-defined objects(Value Objects)

By representing data as well-defined objects(value objets), i.e. By creating class out of them, we can avoid a few issues when designing APIs.
In practical terms you can think of value objects as your own primitive types.

When should value objects be used? 

Basically every time we have some value that has a special meaning in our business logic, even when it could be represented by a primitive type and especially if there are some limitations on the value it’d hold.

eg
```
final class Currency
{
    /** @var string */
    private $currency;
 
    public function __construct(string $currency)
    {
        $currency = strtoupper($currency);
        if (strlen($currency) !== 3 || !ctype_alpha($currency)) {
            throw new InvalidArgumentException("Currency has to consist of three letters");
        }
        $this->currency = $currency;
    }
 
    // Rest of the definition should be here
}
```

How do we use it

```
Currency $currency;

or

public function add(Currency $currency)
{
    
}
```
By doing above we validates Currency before accepting the value as input.
There are cases in our current api where we can apply value objects.
eg `donor_id`, `donee_id` etc

## 3. Use forward compatible attribute naming

When naming attributes of objects in your API responses be sure to name
them in such a manner that they’re going to be forward compatible with 
any updates you’re planning on making in the future. One of the worst 
things we can do to an API is to release a backwards-breaking change. 
As a rule of thumb, adding new fields to an object doesn’t break compatibility. 
Clients can simply choose to ignore new fields. Changing the type, or removing a field, will break clients and must be avoided.

eg

When home town is not made an object at the beginning
```
{
  "name": "Thomas Hunter II",
  "username": "tlhunter",
  "hometown": "Ann Arbor"
}
```
Adding the following changes will break systems using the prev version

eg 
```
{
  "name": "Thomas Hunter II",
  "username": "tlhunter",
  "hometown": {
    "city": "Ann Arbor",
    "municipality": "MI"
  }
}

```

## 4. Use HATEOAS

Hypermedia as the Engine of Application State is a principle that hypertext links
 should be used to create a better navigation through the API.

```
{
  "id": 711,
  "manufacturer": "bmw",
  "model": "X5",
  "seats": 5,
  "drivers": [
   {
    "id": "23",
    "name": "Stefan Jauker",
    "links": [
     {
     "rel": "self",
     "href": "/api/v1/drivers/23"
    }
   ]
  }
 ]
}
```

This enables application consumming the api be able to know how to locate certain resources dynamically
with hardcoding the path.


## 5.  GET method and query parameters should not alter the state
Use PUT, POST and DELETE methods  instead of the GET method to alter the state.
Do not use GET for state changes:

GET /users/711?activate or
GET /users/711/activate


## 5. Handle Errors with HTTP status codes
Use HTTP status codes

200 – OK – Eyerything is working
201 – OK – New resource has been created
204 – OK – The resource was successfully deleted

304 – Not Modified – The client can use cached data

400 – Bad Request – The request was invalid or cannot be served. 
The exact error should be explained in the error payload. E.g. „The JSON is not valid“

401 – Unauthorized – The request requires an user authentication

403 – Forbidden – The server understood the request, 
but is refusing it or the access is not allowed.

404 – Not found – There is no resource behind the URI.

422 – Unprocessable Entity – Should be used if the server cannot process the enitity, 
e.g. if an image cannot be formatted or mandatory fields are missing in the payload.

500 – Internal Server Error – API developers should avoid this error. If an error occurs in the global catch blog, the stracktrace should be logged and not returned as response.
