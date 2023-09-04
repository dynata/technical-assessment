# Introduction
For this technical assessment you will be asked to write an implementation of a custom protocol for signing HTTP requests.

You may write your implementation using any programming language that you are comfortable with as well as any external libraries that are available to you. The only requirements for the design of your interface are as follows:

The interface must accept 3 inputs:

- `AccessKey` as a string value.
- `SecretKey` as a string value.
- an HTTP request frame as a string (or byte array/stream) value.

The interface must output 1 string value that is the calculated `Signature` for the given inputs.

Do not spend too much time on this assessment. If you find yourself working more than a couple of hours, it is absolutely acceptable to come to your interview without a working solution. The primary goal is to display how you approach the solution to a problem, as well as your coding style.

Your solution implementation will be evaluated based on the following attributes:

- **Efficiency**: looking at time and space complexity of implementation.
- **Completeness**: looking for proper handling of exceptional or edge cases.
- **Correctness**: looking for expected output for a given input. (can be shown via unit tests)
- **Readability**: looking for concise, self-describing code. (relevant comments are welcome)
- **Simplicity**: looking for a simple interface that communicates all necessary information to the client. (encapsulation of complexity)
- **Ergonomics**: looking for an interface that makes sense and is easy to use.

Your implementation should attempt to handle as many input cases as possible, which may include invalid inputs. When possible, attempt to handle invalid inputs in a reasonable way, otherwise return an error through your interface. For instance, if a given HTTP request frame URI has a query string containing a reserved character that is not percent encoded, you may ignore it on input and encode it properly yourself, when applicable according to the signing protocol, and move on.

Once you have finished your assessment, please have it ready in a way that can be shared with your interviewer(s). Preferably, upload your solution to your personal GitHub (or similar) account as a repository. We will go over your code together during your technical interview.

If you have any questions about the assessment, please do not hesitate to email the recruiter that has been working with you up to this point. They will ensure that an answer gets back to you from the technical team.

# HTTP Request Signing Protocol
Following is a protocol definition for signing HTTP request frames. This protocol is capable of signing any HTTP/1.1 request frame that is not using a transfer encoding of "chunked".

The following table describes the functions that are used in the definition of the signing protocol. You will need to implement code for these functions or use equivalent functions from one or more libraries.

| Function                | Description                                                                                                   |
| ----------------------- | ------------------------------------------------------------------------------------------------------------- |
| Lowercase(value)        | Converts the string to lowercase.                                                                             |
| Hex(value)              | Base 16 encodes the given value.                                                                              |
| SHA256(value)           | Calculates the hash digest of the given value using the 256 bit version of the Secure Hashing Algorithm (SHA) |
| HMAC-SHA256(key, value) | Computes an HMAC of the given value by using the SHA256 algorithm with the signing key provided.              |
| Trim(value)             | Removes any leading or trailing whitespace from the given value.                                              |
| UriEncode(value)        | URI encodes every byte of the given value.                                                                    |

### URI encoding

URI encoding must enforce the following rules:
- URI encode every byte except the unreserved characters: 'A'-'Z', 'a'-'z', '0'-'9', '-', '.', '_', and '~'.
- The space character is a reserved character and must be encoded as "%20" (and not as "+").
- Each URI encoded byte is formed by a '%' and the two-digit hexadecimal value of the byte.
- Letters in the hexadecimal value must be uppercase, for example "%1A".

> :warning: The standard UriEncode functions provided by your development platform may not work because of differences
> in implementation and related ambiguity in the underlying RFCs. We recommend that you write your own custom UriEncode
> function to ensure that your encoding will work. Most implementations that follow RFC 3986 will suffice.

## Step 1: Create a Canonical Request
The following is the canonical request format used to calculate a signature:
```text
<HTTPMethod>\n
<CanonicalURI>\n
<CanonicalQueryString>\n
<CanonicalHeaders>\n
<HashedPayload>
```
Where:

`HTTPMethod` is a valid HTTP method as defined by RFC 7231 section 4.3

`CanonicalURI` is the URI-encoded version of the absolute path component of the URI; everything starting with the forward-slash '/' that follows the domain name and up to the end of the string or to the question mark character '?'. Do not normalize the URI path. For instance, do not change "hello//world" to "hello/world".

`CanonicalQueryString` specifies the URI-encoded query string parameters.
- Repeated parameter names must be combined into a single parameter before encoding. Concatenate their values into a single string value, separated by commas. Values must be concatenated in the order in which the parameters appear within the query string.
- URI-encode names and values individually. 
- After encoding, sort the parameters in the canonical query string by parameter name, in ascending order of character code point. For example, a parameter name that begins with the uppercase letter F precedes a parameter name that begins with a lowercase letter b.

An example request encoded query string:
```text
prefix=somePrefix&marker=some+marker&max-keys=20
```
After decoding, would be processed as:
```text
UriEncode("marker")+"="+UriEncode("some marker")+"&"+
UriEncode("max-keys")+"="+UriEncode("20")+"&"+
UriEncode("prefix")+"="+UriEncode("somePrefix")
```
Resulting in a canonical query string of:
```text
marker=some%20marker&max-keys=20&prefix=somePrefix
```
If the URI does not include any query string parameters, then you will set the canonical query string to an empty string ("").

`CanonicalHeaders` is a list of request headers with their values. 
- Individual header field name and value pairs are separated by the newline character ("\n"). 
- Header field names must use lowercase characters and must be followed by a colon (":"). 
- For the header field values, trim any leading or trailing spaces. 
- Sort the header field names in ascending order of character code point.
- Repeated header field names must be combined into a single header field. Concatenate their values into a single string value, separated by commas. Values must be concatenated in the order in which the fields appear within the header.

The following example shows the format:
```text
Lowercase(<HeaderName1>)+":"+Trim(<value>)+"\n"
Lowercase(<HeaderName2>)+":"+Trim(<value>)+"\n"
...
Lowercase(<HeaderNameN>)+":"+Trim(<value>)+"\n"
```
The `Lowercase()` and `Trim()` functions used in this example are described in the preceding table of functions.

`HashedPayload` is the hexadecimal value of the SHA256 hash digest of the request payload.
```text
Lowercase(Hex(SHA256(<payload>)))
```
If there is no payload in the request, you compute a hash digest of the empty string as follows:
```text
Lowercase(Hex(SHA256("")))
```
The hash digest of the empty string will return the following value:
```text
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

## Step 2: Create a String to Sign
The string to sign is a SHA256 hash digest of the canonical request. It is calculated as follows:
```text
StringToSign = Lowercase(Hex(SHA256(<CanonicalRequest>)))
```

## Step 3: Calculate the Signature
Instead of using the `SecretKey` to sign the request directly, we will first create a derived key made up of the `SecretKey` the current date in the format `YYYYMMDD` and the `AccessKey`. To produce the `SigningKey`, follow the below algorithm:
```text
DateKey    = Lowercase(Hex(HMAC-SHA256(<SecretKey>, <YYYYMMDD>)))
SigningKey = Lowercase(Hex(HMAC-SHA256(<DateKey>, <AccessKey>)))
```
The signature can then be calculated using the `StringToSign` and the `SigningKey`:
```text
Signature = Lowercase(Hex(HMAC-SHA256(<SigningKey>, <StringToSign>)))
```
This `Signature` will be the output of your interface.

## Examples
Below are a series of examples that you can use to test your implementation. The examples provided are limited and are not a full representation of all input cases.

All examples will use the following `AccessKey` and `SecretKey`:
```text
AccessKey = AKIAIOSFODNN7EXAMPLE
SecretKey = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```
To keep things deterministic, the following date value will be used:
```text
YYYYMMDD = 20230801
```

### Signing Key
Since you will be using the same constant values to produce the signing key, the calculated hash is provided here to aide in troubleshooting your own work:
```test
28cfe47c386456f844def6a497e09cb7de1a52569bd65449792938acf550ca34
```

### Example 1:
#### HTTP Frame
```text
GET /resource?test=true&mix=1%C2%B11 HTTP/1.1\r\n
Host: test.com\r\n
Timestamp: 2023-08-03T10:24:03.012Z\r\n
\r\n
```

#### Canonical Request
```text
GET\n
/resource\n
mix=1%C2%B11&test=true\n
host:test.com\n
timestamp:2023-08-03T10:24:03.012Z\n
\n
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

#### String To Sign
```text
0c5a284aca21eb658a5e4503729f23c47a648e6acaf4613e917d3406e6e8cec0
```

#### Signature
```text
48c48534128e1603216519035b52821c1c945c563f4d06031369b0552396635e
```

### Example 2:
#### HTTP Frame
```text
POST /resource//posts?test=2&example=all+please&1234=4321&test=1 HTTP/1.1\r\n
Host: test.com\r\n
Choice: C\r\n
Choice: A\r\n
Choice: B\r\n
Content-Length: 25\r\n
\r\n
{"some_key":"some_value"}
```

#### Canonical Request
```test
POST\n
/resource//posts\n
1234=4321&example=all%20please&test=2%2C1\n
choice:C,A,B\n
content-length:25\n
host:test.com\n
\n
43ee763040973ca602549c94c5357a41c280afbb54e48d436af88f4e40d73081
```

#### String To Sign
```test
28fe39b38e2590cbc242dd417f604ca4a6fe91fd9572d8f47520efedd49670e0
```

#### Signature
```test
18e53de99fb8cf5824fc879336a12927dcf7f6d7c42607f87a02a13f690134b1
```

### Example 3:
#### HTTP Frame
```text
POST /resource/123/comments?test=%2TRUE%&evaluation=1±2 HTTP/1.1\r\n
Host: test.com\r\n
Content-Length: 65\r\n
Content-Type: text/plain\r\n
\r\n
this is a sentence.\r\nthis sentence is on a new line.\r\nso is this.
```

#### Canonical Request
```text
POST\n
/resource/123/comments\n
evaluation=1%C2%B12&test=%252TRUE%25\n
content-length:65\n
content-type:text/plain\n
host:test.com\n
\n
a41088b4f429f0a67ce7c5a2b8507d24ae475f3a93e0840a0986fc4e45ff0487
```

#### String To Sign
```text
5fc252d8fa4e4335f8e22b6109a59922858c2c0b3e71405096ea2fd5f68c667b
```

#### Signature
```text
b73c62f23924c051464a4342ed26389c9e68182a8601c701820c5155d4acbb22
```
