# Introduction #

While the core, required methods of the OAuth2 library are marked as abstract, several others are only required to support particular grant types.

Marking all of these methods abstract would require subclasses to implement all of them, which is undesirable, because they would not all be used.

Depending on the grant types that your OAuth2 library supports, you will need to implement the methods detailed below.

# Authorization Codes #

## Required methods ##

  * get\_stored\_auth\_code($code)
  * store\_auth\_code($code, $client\_id, $redirect\_uri, $expires, $scope)

These methods store and retrieve authorization codes, using persistent storage.

### get\_stored\_auth\_code ###

Given a code, return an array:

```
 array (
  "client_id" => <stored client id>,
  "redirect_uri" => <stored redirect URI>,
  "expires" => <stored code expiration time>,
  "scope" => <stored scope values (space-separated string), or can be omitted if scope is unused>
 )
```

Return **null** if the code is invalid.

### store\_auth\_code ###

Take the given values and store them in your database.  All values except for $expires are supplied as strings.  $expires is an integer (UNIX timestamp).

_If storage fails, you should throw a descriptive message and exit._  The library does not check the return value for success or failure.

# Basic User Credentials #

## Required methods ##

  * check\_user\_credentials($client\_id, $username, $password)

### check\_user\_credentials ###

Verify the given username and password (also called a shared secret).  Return **false** if the credentials are invalid.

If your resources require a given scope, you should return an array that contains the scope of the user's access:

```
 array (
  "scope" => <user's scope values (space-separated string)>
 )
```

# Assertions #

## Required methods ##

  * check\_assertion($client\_id, $assertion\_type, $assertion)

### check\_assertion ###

Verify the given assertion.  Return **false** if the assertion is invalid.

If your resources require a given scope, you should return an array that contains the scope of the assertion's access:

```
 array (
  "scope" => <scope values from the assertion (space-separated string)>
 )
```

# Refresh Tokens #

## Required methods ##

  * get\_refresh\_token($refresh\_token)
  * store\_refresh\_token($token, $client\_id, $expires, $scope = null)

### get\_refresh\_token ###

Given a refresh token id, retrieve the token from storage and return an array with the values:

```
 array (
  "client_id" => <stored client id>,
  "expires" => <stored code expiration time>,
  "scope" => <stored scope values (space-separated string), or can be omitted if scope is unused>
 )
```

### store\_refresh\_token ###

Take the given values and store them in your database.  All values except for $expires are supplied as strings.  $expires is an integer (UNIX timestamp).

_If storage fails, you should throw a descriptive message and exit._  The library does not check the return value for success or failure.

# None #

## Required methods ##

  * check\_none\_access($client\_id)

### check\_none\_access ###

The spec doesn't really describe this access method very well yet.  Return true to grant access or false to deny access.

If your resources require a given scope, you should return an array that contains the access scope:

```
 array (
  "scope" => <scope values from the assertion (space-separated string)>
 )
```