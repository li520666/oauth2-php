# Introduction #

To demonstrate the OAuth 2.0 server library, we built a sample implementation using Mongo DB for persistent storage.

Chances are good that you're not using Mongo, so you'll want to have a library for your database (MySQL, etc.).  Let's step through how we made the Mongo implementation.

Don't get too hung up on the Mongo syntax.  If you're unfamiliar with Mongo, it easily stores and retrieves PHP arrays.  That's a gross oversimplification; it does **much** more, but it's enough for you to understand what's going on.

If you're not familiar with OAuth 2.0 at all, you would benefit from reading the http://tools.ietf.org/html/draft-ietf-oauth-v2 IETF draft].

# Getting Started #

## Defining our Mongo collections ##

We will be using three collections to hold our data.  Each Mongo collection is roughly analogous to a MySQL table.

  * **auth\_codes** will hold our authorization codes, temporary codes used to gain authorization from a user to get an access token.
  * **clients** will hold our client definitions.
  * **tokens** will hold our access tokens, used to gain access to protected resources.

## Creating the Mongo subclass ##

We start by extending the OAuth2 class.  Our file looks like this:
```

include "oauth.php";

class MongoOAuth2 extends OAuth2 {

}

```

## Connecting to our database ##

Now, we want to connect to our Mongo database when the class is created, so we add the following:
```

define("MONGO_CONNECTION", "mongodb://user:pass@mongoserver/mydb");
define("MONGO_DB", "mydb");

include "oauth.php";

class MongoOAuth2 extends OAuth2 {
    private $db;

    public function __construct() {
        parent::__construct();

        $mongo = new Mongo(MONGO_CONNECTION);
        $this->db = $mongo->selectDB(MONGO_DB);
    }
}

```

Now when a class instance is created, it will connect to our Mongo DB.  The $db member variable holds our database connection for later use.

## Implementing the abstract methods ##

The OAuth2 class defines several abstract methods that all subclasses must implement.

  * **abstract protected function auth\_client\_credentials($client\_id, $client\_secret = null);**
> Take a registered client's id and, optionally, a shared secret (password).  Return true if the credentials are valid and false if not.
  * **abstract protected function get\_redirect\_uri($client\_id);**
> Look up the registered client by id and return the redirect\_uri that they registered with.
  * **abstract protected function get\_access\_token($token\_id);**
> Look up the access token and return an array with its details.  Return null if the token is not valid.  The return array is:
```
     array("client_id" => stored client id, 
           "expires" => stored expiration timestamp (int),
           "scope" => stored access token scope (string, may be null)
     )
```
  * **abstract protected function store\_access\_token($token\_id, $client\_id, $expires, $scope = null);**
> Commit the provided access token values to storage.  Basically, save these things to your database.

Implementing these in our Mongo class, now it looks like:

```

class MongoOAuth2 extends OAuth2 {
    private $db;

    public function __construct() {
        parent::__construct();

        $mongo = new Mongo(MONGO_CONNECTION);
        $this->db = $mongo->selectDB(MONGO_DB);
    }

    // Do NOT use this in production!  This sample code stores the secret in plaintext!
    protected function auth_client_credentials($client_id, $client_secret = null) {
        $client = $this->db->clients->findOne(array("_id" => $client_id, "pw" =>  $client_secret));
        return $client !== null;
    }

    protected function get_redirect_uri($client_id) {
        $uri = $this->db->clients->findOne(array("_id" => $client_id), array("redirect_uri"));
        return $uri !== null ? $uri["redirect_uri"] : null;
    }

    protected function get_access_token($token_id) {
        return $this->db->tokens->findOne(array("_id" => $token_id));
    }

    protected function store_access_token($token_id, $client_id, $expires, $scope = null) {
        $this->db->tokens->insert(array(
            "_id" => $token_id,
            "client_id" => $client_id,
            "expires" => $expires,
            "scope" => $scope
        ));
    }
}
```

## Implementing "optional" functionality ##

We've just finished implementing all of the **required** methods.  However, our subclass still won't do anything useful.

We need to tell it what "access grant types" we support.  Basically, an access grant type is a way for your OAuth client to gain an access token.  So, how can our OAuth clients access our resources?

The OAuth 2.0 spec defines several access grant types:
  * Authorization Codes
  * Basic User Credentials (username and password)
  * Assertion
  * Refresh Tokens
  * None

Descriptions of these grant types are outside the scope of this document.  For this example, we're going to implement the **Authorization Code** grant type, so that OAuth clients can use auth codes to gain access to protected resources.

### Define supported grant types ###

We need to override the **get\_supported\_grant\_types** method to indicate that we're supporting the auth code grant type.  Our new method looks like:

```
    protected function get_supported_grant_types() {
        return array(AUTH_CODE_GRANT_TYPE);
    }
```

### Store and retrieve authorization codes ###

To work with auth codes, we need to tell the OAuth2 library how to store and retrieve them.  We do this by overriding the **get\_stored\_auth\_code** and **store\_auth\_code** methods.  These overrides look like:

```
    protected function get_stored_auth_code($code) {
        $stored_code = $this->db->auth_codes->findOne(array("_id" => $code));
        return $stored_code !== null ? $stored_code : false;
    }

    protected function store_auth_code($code, $client_id, $redirect_uri, $expires, $scope) {
        $this->db->auth_codes->insert(array(
            "_id" => $code,
            "client_id" => $client_id,
            "redirect_uri" => $redirect_uri,
            "expires" => $expires,
            "scope" => $scope
        ));
    }
```

Now our class looks like this:

```
class MongoOAuth2 extends OAuth2 {
    private $db;

    public function __construct() {
        parent::__construct();

        $mongo = new Mongo(MONGO_CONNECTION);
        $this->db = $mongo->selectDB(MONGO_DB);
    }

    // Do NOT use this in production!  This sample code stores the secret in plaintext!
    protected function auth_client_credentials($client_id, $client_secret = null) {
        $client = $this->db->clients->findOne(array("_id" => $client_id, "pw" =>  $client_secret));
        return $client !== null;
    }

    protected function get_redirect_uri($client_id) {
        $uri = $this->db->clients->findOne(array("_id" => $client_id), array("redirect_uri"));
        return $uri !== null ? $uri["redirect_uri"] : null;
    }

    protected function get_access_token($token_id) {
        return $this->db->tokens->findOne(array("_id" => $token_id));
    }

    protected function store_access_token($token_id, $client_id, $expires, $scope = null) {
        $this->db->tokens->insert(array(
            "_id" => $token_id,
            "client_id" => $client_id,
            "expires" => $expires,
            "scope" => $scope
        ));
    }

    protected function get_supported_grant_types() {
        return array(AUTH_CODE_GRANT_TYPE);
    }

    protected function get_stored_auth_code($code) {
        $stored_code = $this->db->auth_codes->findOne(array("_id" => $code));
        return $stored_code !== null ? $stored_code : false;
    }

    protected function store_auth_code($code, $client_id, $redirect_uri, $expires, $scope) {
        $this->db->auth_codes->insert(array(
            "_id" => $code,
            "client_id" => $client_id,
            "redirect_uri" => $redirect_uri,
            "expires" => $expires,
            "scope" => $scope
        ));
    }
}
```

## Adding extra functionality ##

For the example, we thought it would be useful for our Mongo class to also insert simple client records into the client collection.  Our method looks like this:

```

    public function add_client($client_id, $secret, $redirect_uri) {
        $this->db->clients->insert(array(
            "_id" => $client_id,
            "pw" => $secret,
            "redirect_uri" => $redirect_uri
        ));
    }

```

Add it all up, and you've got the example Mongo library!  Hope you had fun!

**Don't forget to check the source code repo for a sample server implementation using the Mongo library!**