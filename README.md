# Introduction
For this technical assessment you will be asked to write an implementation of a custom protocol for signing HTTP requests.

You may write your implementation using any programming language that you are comfortable with. The only requirements for the design of your interface are as follows:

The interface must accept 3 inputs:

- `AccessKey` as a string value.
- `SecretKey` as a string value.
- an HTTP request frame as a string (or byte array/stream) value.

The interface must output 1 string value that is the calculated `Signature` for the given inputs.

Do not spend too much time on this assessment. If you find yourself working more than a couple of hours, it is absolutely acceptable to come to your interview without a working solution. The primary goal is to display how you approach the solution to a problem, as well as your coding style.

Once you have finished your assessment, please have it ready in a way that can be shared with your interviewer(s). Preferably, upload your solution to your personal GitHub (or similar) account as a repository. We will go over your code together during your technical interview.

If you have any questions about the assessment, please do not hesitate to email the recruiter that has been working with you up to this point. They will ensure that an answer gets back to you from the technical team.

# HTTP Request Signing Protocol
The following table describes the functions that are used in the definition of the signing protocol.
You will need to implement code for these functions or use equivalent functions from one or more libraries.

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

`CanonicalQueryString` specifies the URI-encoded query string parameters. You URI-encode names and values individually. After encoding, you must also sort the parameters in the canonical query string by parameter name in ascending order of character code point. For example, a parameter name that begins with the uppercase letter F precedes a parameter name that begins with a lowercase letter b.

An example query string:
```text
prefix=somePrefix&marker=some+marker&max-keys=20
```
Would be processed as:
```text
UriEncode("marker")+"="+UriEncode("some marker")+"&"+
UriEncode("max-keys")+"="+UriEncode("20") + "&" +
UriEncode("prefix")+"="+UriEncode("somePrefix")
```
Resulting in a canonical query string of:
```text
marker=some%20marker&max-keys=20&prefix=somePrefix
```
If the URI does not include any query string parameters, then you will set the canonical query string to an empty string ("").

`CanonicalHeaders` is a list of request headers with their values. Individual header field name and value pairs are separated by the newline character ("\n"). Header field names must use lowercase characters and must be followed by a colon (":"). For the header field values, trim any leading or trailing spaces and separate the values for a multi-value header using commas. You must sort the header field names in ascending order of character code point. The following example shows the format:
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
GET /resource?test=true&mix=1%C2%B11 HTTP/1.1
Host: test.com
Timestamp: 2023-08-03T10:24:03.012Z

```

#### Canonical Request
```text
GET
/resource
mix=1%C2%B11&test=true
host:test.com
timestamp:2023-08-03T10:24:03.012Z

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
POST /resource//posts?test=true&example=all+please&1234=4321 HTTP/1.1
Host: test.com
Choice: A
Choice: B
Choice: C
Content-Length: 25

{"some_key":"some_value"}
```

#### Canonical Request
```test
POST
/resource//posts
1234=4321&example=all%20please&test=true
choice:A,B,C
content-length:25
host:test.com

43ee763040973ca602549c94c5357a41c280afbb54e48d436af88f4e40d73081
```

#### String To Sign
```test
ee483f641c7ee1da1b2ec2eaa897885f24305bd362a190a0451f2b5d1df3e683
```

#### Signature
```test
ca365f9f9b8c6f327d26b8f7ef69cb779b8434eede34b430192de20dd799d755
```
